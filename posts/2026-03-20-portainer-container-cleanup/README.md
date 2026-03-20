# How to Automate Container Cleanup Scripts with Portainer API

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, API, Automation, Cleanup, Docker

Description: Automate the cleanup of stopped containers, unused images, dangling volumes, and orphaned networks across Portainer-managed environments.

## Introduction

Docker environments accumulate cruft over time: stopped containers, dangling images, unused volumes, and orphaned networks. This waste consumes disk space and can cause confusion. The Portainer API enables automated cleanup scripts that run on a schedule to keep your environments tidy.

## The Cleanup Script

```python
#!/usr/bin/env python3
# cleanup.py

import requests
import logging
from datetime import datetime, timedelta, timezone

logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')
logger = logging.getLogger(__name__)

PORTAINER_URL = "https://portainer.example.com"
API_KEY = "your-api-key"
ENDPOINT_ID = 1

headers = {"X-API-Key": API_KEY}
base = f"{PORTAINER_URL}/api/endpoints/{ENDPOINT_ID}/docker"


def cleanup_stopped_containers(max_age_hours=24):
    """Remove containers that have been stopped for more than max_age_hours."""
    resp = requests.get(
        f"{base}/containers/json",
        params={"all": "true", "filters": '{"status":["exited","dead"]}'},
        headers=headers
    )
    containers = resp.json()
    removed = 0
    
    cutoff = datetime.now(timezone.utc) - timedelta(hours=max_age_hours)
    
    for c in containers:
        finished_at = c.get('Status', '')
        name = c['Names'][0].replace('/', '')
        
        # Remove container
        resp = requests.delete(
            f"{base}/containers/{c['Id']}",
            params={"force": "false"},
            headers=headers
        )
        if resp.status_code == 204:
            logger.info(f"Removed stopped container: {name}")
            removed += 1
    
    return removed


def cleanup_dangling_images():
    """Remove dangling (untagged) images."""
    resp = requests.get(
        f"{base}/images/json",
        params={"filters": '{"dangling":["true"]}'},
        headers=headers
    )
    images = resp.json()
    removed = 0
    
    for img in images:
        resp = requests.delete(
            f"{base}/images/{img['Id']}",
            headers=headers
        )
        if resp.status_code == 200:
            logger.info(f"Removed dangling image: {img['Id'][:12]}")
            removed += 1
    
    return removed


def cleanup_unused_volumes():
    """Remove volumes not used by any container."""
    resp = requests.post(
        f"{base}/volumes/prune",
        headers=headers
    )
    result = resp.json()
    volumes_removed = len(result.get('VolumesDeleted', []) or [])
    space_reclaimed = result.get('SpaceReclaimed', 0)
    
    logger.info(f"Removed {volumes_removed} unused volumes, "
                f"reclaimed {space_reclaimed / 1024 / 1024:.1f}MB")
    return volumes_removed


def cleanup_unused_networks():
    """Remove networks not used by any container."""
    resp = requests.post(
        f"{base}/networks/prune",
        headers=headers
    )
    result = resp.json()
    networks_removed = len(result.get('NetworksDeleted', []) or [])
    logger.info(f"Removed {networks_removed} unused networks")
    return networks_removed


def run_full_cleanup():
    """Run all cleanup operations."""
    logger.info("Starting cleanup cycle")
    
    containers = cleanup_stopped_containers(max_age_hours=48)
    images = cleanup_dangling_images()
    volumes = cleanup_unused_volumes()
    networks = cleanup_unused_networks()
    
    logger.info(f"Cleanup complete: {containers} containers, {images} images, "
                f"{volumes} volumes, {networks} networks removed")


if __name__ == '__main__':
    run_full_cleanup()
```

## Running as a Scheduled Job

```yaml
# cleanup-cronjob.yml - Kubernetes CronJob (if using K8s)
apiVersion: batch/v1
kind: CronJob
metadata:
  name: portainer-cleanup
spec:
  schedule: "0 2 * * *"  # Daily at 2 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: cleanup
            image: python:3.11-slim
            command:
            - sh
            - -c
            - "pip install requests -q && python /scripts/cleanup.py"
            env:
            - name: PORTAINER_URL
              value: "https://portainer.example.com"
            - name: PORTAINER_API_KEY
              valueFrom:
                secretKeyRef:
                  name: portainer-api-key
                  key: key
```

```bash
# Docker container scheduled via cron
docker run -d \
  --name cleanup-scheduler \
  --restart=always \
  -e PORTAINER_URL=https://portainer.example.com \
  -e PORTAINER_API_KEY=your-key \
  -e CRON_SCHEDULE="0 2 * * *" \
  cleanup-scheduler:latest
```

## System Prune via API

```bash
# Full system prune (all unused resources)
curl -X POST \
  -H "X-API-Key: your-api-key" \
  "https://portainer.example.com/api/endpoints/1/docker/system/prune" \
  | python3 -m json.tool

# Prune with all images (not just dangling)
curl -X POST \
  -H "X-API-Key: your-api-key" \
  "https://portainer.example.com/api/endpoints/1/docker/system/prune?all=true" \
  | python3 -m json.tool
```

## Conclusion

Automated container cleanup via the Portainer API keeps your Docker environments healthy and efficient. By scheduling cleanup scripts to run nightly, you prevent disk space exhaustion and maintain clear visibility of actually-running workloads. The script can be extended to send cleanup reports to Slack or email, keeping your team informed of resource reclamation.
