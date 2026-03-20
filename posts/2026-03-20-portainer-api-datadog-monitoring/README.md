# How to Integrate Portainer API with Datadog for Monitoring - Monitoring

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Datadog, Monitoring, API, Observability, DevOps

Description: Use the Portainer REST API to collect container and service metrics, then submit them to Datadog for dashboards, anomaly detection, and alerting across your container infrastructure.

---

Portainer exposes container statistics and service health through its REST API. By periodically querying this API and forwarding the data to Datadog's metrics API, you can build dashboards and alerts for container infrastructure without deploying the Datadog agent on every host.

## Integration Approaches

| Approach | When to Use |
|---|---|
| Portainer API → Datadog Metrics API | Simple metrics from Portainer's perspective |
| Datadog Agent on hosts | Full host + container metrics, auto-discovery |
| OpenTelemetry Collector | If you also have application traces/logs |

This guide covers the Portainer API approach for environments where deploying the Datadog agent is not feasible.

## Step 1: Get Portainer API Token

In Portainer, go to **My Account > Access Tokens** and generate a token.

## Step 2: Get Datadog API Key

In Datadog, go to **Organization Settings > API Keys** and create an API key.

## Step 3: Metrics Collection Script

```python
#!/usr/bin/env python3
"""
Collect Portainer container stats and forward to Datadog.
"""
import requests
import time
import os

PORTAINER_URL = os.environ["PORTAINER_URL"]
PORTAINER_TOKEN = os.environ["PORTAINER_TOKEN"]
DATADOG_API_KEY = os.environ["DD_API_KEY"]
DATADOG_SITE = os.environ.get("DD_SITE", "datadoghq.com")
ENDPOINT_ID = int(os.environ.get("PORTAINER_ENDPOINT_ID", "1"))

def get_containers():
    resp = requests.get(
        f"{PORTAINER_URL}/api/endpoints/{ENDPOINT_ID}/docker/containers/json",
        headers={"Authorization": f"Bearer {PORTAINER_TOKEN}"},
        params={"all": "false"}  # Only running containers
    )
    resp.raise_for_status()
    return resp.json()

def get_container_stats(container_id):
    resp = requests.get(
        f"{PORTAINER_URL}/api/endpoints/{ENDPOINT_ID}/docker/containers/{container_id}/stats",
        headers={"Authorization": f"Bearer {PORTAINER_TOKEN}"},
        params={"stream": "false"}
    )
    resp.raise_for_status()
    return resp.json()

def calculate_cpu_percent(stats):
    cpu_delta = stats["cpu_stats"]["cpu_usage"]["total_usage"] - \
                stats["precpu_stats"]["cpu_usage"]["total_usage"]
    system_delta = stats["cpu_stats"]["system_cpu_usage"] - \
                   stats["precpu_stats"]["system_cpu_usage"]
    num_cpus = len(stats["cpu_stats"]["cpu_usage"].get("percpu_usage", [1]))
    return (cpu_delta / system_delta) * num_cpus * 100

def submit_to_datadog(metrics):
    now = int(time.time())
    series = [
        {
            "metric": f"portainer.container.{m['name']}",
            "points": [[now, m["value"]]],
            "type": "gauge",
            "tags": m["tags"]
        }
        for m in metrics
    ]
    resp = requests.post(
        f"https://api.{DATADOG_SITE}/api/v1/series",
        headers={
            "Content-Type": "application/json",
            "DD-API-KEY": DATADOG_API_KEY
        },
        json={"series": series}
    )
    return resp.status_code

if __name__ == "__main__":
    containers = get_containers()
    metrics = []
    
    for container in containers:
        cid = container["Id"]
        name = container["Names"][0].lstrip("/")
        
        stats = get_container_stats(cid)
        cpu = calculate_cpu_percent(stats)
        memory = stats["memory_stats"]["usage"]
        memory_limit = stats["memory_stats"]["limit"]
        
        tags = [f"container:{name}", f"image:{container['Image']}"]
        
        metrics.extend([
            {"name": "cpu_percent", "value": cpu, "tags": tags},
            {"name": "memory_bytes", "value": memory, "tags": tags},
            {"name": "memory_percent", "value": (memory / memory_limit) * 100, "tags": tags}
        ])
    
    status = submit_to_datadog(metrics)
    print(f"Submitted {len(metrics)} metrics to Datadog: HTTP {status}")
```

## Step 4: Deploy as a Portainer Stack

```yaml
version: "3.8"
services:
  datadog-bridge:
    image: python:3.12-slim
    command: >
      sh -c "pip install requests -q &&
             while true; do python /app/collector.py; sleep 30; done"
    environment:
      - PORTAINER_URL=https://portainer:9443
      - PORTAINER_TOKEN=${PORTAINER_TOKEN}
      - DD_API_KEY=${DD_API_KEY}
      - DD_SITE=datadoghq.com
    volumes:
      - ./collector.py:/app/collector.py:ro
    restart: unless-stopped
```

## Step 5: Create Datadog Dashboard

With metrics flowing, create a Datadog dashboard with:

- Time series: `portainer.container.cpu_percent` by container tag
- Top list: highest memory consumers
- Alert: CPU > 80% for 5 minutes

## Summary

Portainer's REST API provides container metrics that can be forwarded to Datadog for centralized monitoring. This approach works without installing the Datadog agent directly on hosts, making it useful for environments with strict agent installation requirements.
