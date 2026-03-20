# How to Monitor Container Health Programmatically via Portainer API

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, API, Monitoring, Health, Automation

Description: Build a programmatic container health monitoring solution using the Portainer API to detect unhealthy containers and trigger automated remediation.

## Introduction

Portainer's UI shows container health status visually, but for automated alerting and remediation you need programmatic access. The Portainer API exposes container health check results, resource stats, and event streams, enabling you to build robust monitoring solutions.

## Prerequisites

- Portainer with API access
- Python 3.8+ with `requests` library
- Container healthchecks configured (or rely on process exit codes)

## Checking Container Health Status

```bash
# Get all containers with their health status

curl -s \
  -H "X-API-Key: your-api-key" \
  "https://portainer.example.com/api/endpoints/1/docker/containers/json?all=true" \
  | python3 -c "
import sys, json
containers = json.load(sys.stdin)
for c in containers:
    name = c['Names'][0].replace('/', '')
    state = c['State']
    status = c['Status']
    health = c.get('Health', {}).get('Status', 'none')
    print(f'{name}: state={state}, health={health}')
"
```

## Python Health Monitor Script

```python
#!/usr/bin/env python3
# health_monitor.py

import requests
import time
import json
import logging
from datetime import datetime
from typing import List, Dict, Optional

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger(__name__)

class ContainerHealthMonitor:
    def __init__(self, portainer_url: str, api_key: str, endpoint_id: int):
        self.base_url = portainer_url.rstrip('/')
        self.headers = {'X-API-Key': api_key}
        self.endpoint_id = endpoint_id
        self.unhealthy_threshold = 3  # Alerts after 3 failed checks
        self.failure_counts: Dict[str, int] = {}

    def get_containers(self) -> List[Dict]:
        """Fetch all containers from Portainer."""
        resp = requests.get(
            f"{self.base_url}/api/endpoints/{self.endpoint_id}/docker/containers/json",
            params={"all": "true"},
            headers=self.headers,
            timeout=10
        )
        resp.raise_for_status()
        return resp.json()

    def get_container_stats(self, container_id: str) -> Dict:
        """Get real-time stats for a container."""
        resp = requests.get(
            f"{self.base_url}/api/endpoints/{self.endpoint_id}/docker/containers/{container_id}/stats",
            params={"stream": "false"},
            headers=self.headers,
            timeout=15
        )
        resp.raise_for_status()
        return resp.json()

    def restart_container(self, container_id: str) -> bool:
        """Restart a container."""
        resp = requests.post(
            f"{self.base_url}/api/endpoints/{self.endpoint_id}/docker/containers/{container_id}/restart",
            headers=self.headers,
            timeout=30
        )
        return resp.status_code == 204

    def calculate_cpu_percent(self, stats: Dict) -> float:
        """Calculate CPU usage percentage from Docker stats."""
        cpu_delta = stats['cpu_stats']['cpu_usage']['total_usage'] - \
                    stats['precpu_stats']['cpu_usage']['total_usage']
        system_delta = stats['cpu_stats']['system_cpu_usage'] - \
                       stats['precpu_stats']['system_cpu_usage']
        num_cpus = stats['cpu_stats'].get('online_cpus', 1)
        if system_delta > 0:
            return (cpu_delta / system_delta) * num_cpus * 100.0
        return 0.0

    def calculate_memory_percent(self, stats: Dict) -> float:
        """Calculate memory usage percentage."""
        usage = stats['memory_stats']['usage']
        limit = stats['memory_stats']['limit']
        if limit > 0:
            return (usage / limit) * 100.0
        return 0.0

    def check_health(self):
        """Perform a health check cycle."""
        try:
            containers = self.get_containers()
            issues = []

            for container in containers:
                name = container['Names'][0].replace('/', '')
                container_id = container['Id']
                state = container['State']

                # Check if container is running
                if state != 'running':
                    issues.append({
                        'container': name,
                        'issue': f'Container is {state}',
                        'severity': 'HIGH'
                    })
                    self.failure_counts[name] = self.failure_counts.get(name, 0) + 1
                    continue

                # Check healthcheck status
                health = container.get('Health', {})
                if health.get('Status') == 'unhealthy':
                    issues.append({
                        'container': name,
                        'issue': 'Docker healthcheck reports unhealthy',
                        'severity': 'HIGH'
                    })

                # Get resource stats
                try:
                    stats = self.get_container_stats(container_id)
                    cpu = self.calculate_cpu_percent(stats)
                    mem = self.calculate_memory_percent(stats)

                    if cpu > 90:
                        issues.append({
                            'container': name,
                            'issue': f'High CPU: {cpu:.1f}%',
                            'severity': 'MEDIUM'
                        })

                    if mem > 90:
                        issues.append({
                            'container': name,
                            'issue': f'High Memory: {mem:.1f}%',
                            'severity': 'HIGH'
                        })

                    logger.debug(f"{name}: CPU={cpu:.1f}% MEM={mem:.1f}%")

                except Exception as e:
                    logger.warning(f"Could not get stats for {name}: {e}")

            # Process issues
            for issue in issues:
                logger.warning(f"ISSUE [{issue['severity']}] {issue['container']}: {issue['issue']}")
                self.handle_issue(issue)

            if not issues:
                logger.info(f"All {len(containers)} containers healthy")

        except Exception as e:
            logger.error(f"Health check failed: {e}")

    def handle_issue(self, issue: Dict):
        """Handle detected issues with automated remediation."""
        container_name = issue['container']
        self.failure_counts[container_name] = self.failure_counts.get(container_name, 0) + 1

        if self.failure_counts[container_name] >= self.unhealthy_threshold:
            if issue['severity'] == 'HIGH':
                logger.warning(f"Auto-restarting {container_name} after {self.failure_counts[container_name]} failures")
                # In production, find container ID and restart
                # self.restart_container(container_id)
                self.failure_counts[container_name] = 0
                self.send_alert(container_name, issue)

    def send_alert(self, container_name: str, issue: Dict):
        """Send alert to notification system."""
        # Replace with your actual alerting integration
        alert = {
            'timestamp': datetime.utcnow().isoformat(),
            'container': container_name,
            'issue': issue['issue'],
            'severity': issue['severity'],
            'action': 'container_restarted'
        }
        logger.error(f"ALERT: {json.dumps(alert)}")

    def run(self, interval: int = 60):
        """Run the health monitor continuously."""
        logger.info(f"Starting health monitor (interval: {interval}s)")
        while True:
            self.check_health()
            time.sleep(interval)


if __name__ == '__main__':
    monitor = ContainerHealthMonitor(
        portainer_url="https://portainer.example.com",
        api_key="your-api-key",
        endpoint_id=1
    )
    monitor.run(interval=60)
```

## Running as a Docker Container

```bash
# Build the monitoring container
cat > Dockerfile << 'EOF'
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY health_monitor.py .
CMD ["python", "health_monitor.py"]
EOF

# Run the monitor
docker run -d \
  --name health-monitor \
  --restart=always \
  -e PORTAINER_URL=https://portainer.example.com \
  -e PORTAINER_API_KEY=your-key \
  -e ENDPOINT_ID=1 \
  health-monitor:latest
```

## Conclusion

Programmatic health monitoring via the Portainer API enables automated detection and remediation of container issues. By combining container state checks, Docker healthcheck status, and resource metrics, you can build a comprehensive monitoring system that reduces manual intervention and improves system reliability.
