# How to Set Up Automated Container Health Monitoring with Portainer (2)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Monitoring, Health Check, Automation, Docker

Description: Configure comprehensive automated health monitoring for Docker containers managed by Portainer using built-in health checks and external monitoring integration.

## Introduction

Portainer provides visibility into container health, but proactive monitoring requires automated checks that detect problems before they impact users. This guide covers configuring Docker health checks, setting up external monitoring, and integrating alerts with Portainer's API.

## Step 1: Configure Docker Health Checks

```yaml
# docker-compose.yml with health checks

version: '3.8'

services:
  web:
    image: nginx:latest
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  api:
    image: myapp-api:latest
    healthcheck:
      test: ["CMD-SHELL", "wget -qO- http://localhost:8080/health | grep -q 'ok'"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 60s

  db:
    image: postgres:15
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres -d mydb"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s

  redis:
    image: redis:7
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 3
```

## Step 2: Health Check Endpoint in Applications

```python
# Add a health endpoint to your application
from flask import Flask, jsonify
import psycopg2
import redis

app = Flask(__name__)

@app.route('/health')
def health():
    status = {"status": "ok", "components": {}}
    http_status = 200

    # Check database
    try:
        conn = psycopg2.connect(dsn="...")
        conn.close()
        status["components"]["database"] = "healthy"
    except Exception as e:
        status["components"]["database"] = f"unhealthy: {str(e)}"
        status["status"] = "degraded"
        http_status = 503

    # Check Redis
    try:
        r = redis.Redis(host='redis')
        r.ping()
        status["components"]["redis"] = "healthy"
    except Exception as e:
        status["components"]["redis"] = f"unhealthy: {str(e)}"
        status["status"] = "degraded"

    return jsonify(status), http_status
```

## Step 3: Portainer API Health Monitor

```python
#!/usr/bin/env python3
# health_monitor.py

import requests
import time
import json
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

PORTAINER_URL = "https://portainer.example.com"
API_KEY = "your-api-key"
ENDPOINT_ID = 1
SLACK_WEBHOOK = "https://hooks.slack.com/services/xxx"

headers = {"X-API-Key": API_KEY}


def get_container_health():
    """Get health status of all containers."""
    resp = requests.get(
        f"{PORTAINER_URL}/api/endpoints/{ENDPOINT_ID}/docker/containers/json",
        params={"all": "true"},
        headers=headers
    )
    return resp.json()


def check_health():
    """Check all containers and report issues."""
    containers = get_container_health()
    issues = []

    for c in containers:
        name = c['Names'][0].replace('/', '')
        state = c['State']
        health = c.get('Health', {}).get('Status', 'none')

        if state == 'running' and health == 'unhealthy':
            issues.append({
                'container': name,
                'state': state,
                'health': health,
                'severity': 'HIGH'
            })
        elif state not in ('running', 'created'):
            issues.append({
                'container': name,
                'state': state,
                'health': health,
                'severity': 'CRITICAL'
            })

    if issues:
        send_slack_alert(issues)
    else:
        logger.info(f"All {len(containers)} containers healthy")


def send_slack_alert(issues: list):
    """Send Slack notification for health issues."""
    text = "*Container Health Alert*\n"
    for issue in issues:
        emoji = ":red_circle:" if issue['severity'] == 'CRITICAL' else ":warning:"
        text += f"{emoji} `{issue['container']}`: {issue['state']} / {issue['health']}\n"

    requests.post(SLACK_WEBHOOK, json={"text": text})
    logger.warning(f"Alert sent for {len(issues)} issues")


if __name__ == '__main__':
    logger.info("Starting health monitor")
    while True:
        try:
            check_health()
        except Exception as e:
            logger.error(f"Error: {e}")
        time.sleep(60)
```

## Step 4: Deploy Monitoring as a Portainer Stack

```yaml
# monitoring-stack.yml
version: '3.8'

services:
  health-monitor:
    image: python:3.11-slim
    restart: always
    environment:
      PORTAINER_URL: https://portainer.example.com
      PORTAINER_API_KEY: ${PORTAINER_API_KEY}
      SLACK_WEBHOOK: ${SLACK_WEBHOOK}
    command: >
      sh -c "pip install requests -q && python /app/health_monitor.py"
    volumes:
      - ./health_monitor.py:/app/health_monitor.py:ro
```

## Step 5: Uptime Kuma Integration

```yaml
# uptime-kuma-stack.yml - deploy alongside Portainer
version: '3.8'

services:
  uptime-kuma:
    image: louislam/uptime-kuma:latest
    restart: unless-stopped
    ports:
      - "3001:3001"
    volumes:
      - uptime-kuma-data:/app/data

volumes:
  uptime-kuma-data:
```

Then add monitors in Uptime Kuma:
- Type: HTTP(s)
- URL: `https://portainer.example.com/api/system/status`
- Interval: 60 seconds
- Alert when down for 2 minutes

## Conclusion

Automated container health monitoring with Portainer combines Docker's built-in health checks with external monitoring tools and API integration. This multi-layered approach ensures issues are detected at both the container level (process health) and application level (business logic health), with immediate alerting when problems occur.
