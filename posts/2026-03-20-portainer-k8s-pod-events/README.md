# How to View Pod Events and Logs in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, Logs, Events, Troubleshooting

Description: Access and analyze Kubernetes pod events and container logs using Portainer's web interface for effective troubleshooting.

## Introduction

Pod events and logs are the primary sources of information when troubleshooting Kubernetes workloads. Portainer provides a web-based interface to access these without kubectl, making debugging accessible to developers without cluster CLI access.

## Accessing Logs via Portainer

Navigate to: **Kubernetes > Applications > Your App > Pod > Logs**

Features available in Portainer's log viewer:
- Select which container in a multi-container pod
- Choose previous container instance (after crash)
- Set number of lines to show
- Search/filter log content

## Log Access via API

```bash
# Get logs via Portainer API (proxy to Kubernetes API)

POD_NAME="myapp-pod-xyz"
NAMESPACE="production"

# Current logs
curl -s \
  -H "X-API-Key: your-api-key" \
  "https://portainer.example.com/api/endpoints/1/kubernetes/api/v1/namespaces/$NAMESPACE/pods/$POD_NAME/log?tailLines=100"

# Previous container logs (after crash)
curl -s \
  -H "X-API-Key: your-api-key" \
  "https://portainer.example.com/api/endpoints/1/kubernetes/api/v1/namespaces/$NAMESPACE/pods/$POD_NAME/log?previous=true&tailLines=50"
```

## Viewing Events via Portainer

Navigate to: **Kubernetes > Applications > Your App > Events**

Or for cluster-wide events: **Kubernetes > Cluster > Events**

```bash
# Get events via kubectl for more detail
kubectl get events -n production --sort-by='.lastTimestamp'

# Watch events in real-time
kubectl get events -n production -w

# Get events for a specific pod
kubectl describe pod myapp-pod-xyz -n production | grep -A50 "Events:"

# Events older than the default retention
# Kubernetes by default keeps events for 1 hour
# Use an event aggregator for longer retention
```

## Log Aggregation with Loki

```yaml
# loki-stack.yml - deploy via Portainer for log aggregation
version: '3.8'
services:
  loki:
    image: grafana/loki:latest
    ports:
      - "3100:3100"
    command: -config.file=/etc/loki/local-config.yaml
    volumes:
      - loki-data:/loki

  promtail:
    image: grafana/promtail:latest
    volumes:
      - /var/log:/var/log:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /etc/promtail:/etc/promtail
    command: -config.file=/etc/promtail/config.yml

volumes:
  loki-data:
```

## Structured Logging Best Practices

```python
# Python: structured logging for better Portainer log viewing
import logging
import json

class JSONFormatter(logging.Formatter):
    def format(self, record):
        return json.dumps({
            'timestamp': self.formatTime(record),
            'level': record.levelname,
            'message': record.getMessage(),
            'module': record.module,
            'request_id': getattr(record, 'request_id', None)
        })

logger = logging.getLogger()
handler = logging.StreamHandler()
handler.setFormatter(JSONFormatter())
logger.addHandler(handler)

# Usage
logger.info("Request processed", extra={'request_id': 'req-123'})
```

## Event Analysis Script

```python
#!/usr/bin/env python3
# analyze_events.py

import requests
from collections import Counter
from datetime import datetime

PORTAINER_URL = "https://portainer.example.com"
API_KEY = "your-api-key"

def get_events(namespace: str, endpoint_id: int = 1) -> list:
    resp = requests.get(
        f"{PORTAINER_URL}/api/endpoints/{endpoint_id}/kubernetes/api/v1/namespaces/{namespace}/events",
        headers={"X-API-Key": API_KEY}
    )
    return resp.json().get('items', [])

def analyze_events(namespace: str = "production"):
    events = get_events(namespace)
    
    # Count by type and reason
    by_type = Counter(e['type'] for e in events)
    by_reason = Counter(e['reason'] for e in events if e['type'] == 'Warning')
    
    print(f"Events in {namespace}:")
    print(f"  Normal: {by_type.get('Normal', 0)}")
    print(f"  Warning: {by_type.get('Warning', 0)}")
    
    if by_reason:
        print("\nTop warning reasons:")
        for reason, count in by_reason.most_common(5):
            print(f"  {reason}: {count}")

if __name__ == '__main__':
    analyze_events()
```

## Conclusion

Portainer's pod log and event viewer provides developers direct access to the diagnostic information they need without requiring kubectl access or cluster credentials. For production environments, combine Portainer's built-in log viewing with a log aggregation stack (Loki, Elasticsearch) for searchable, long-term log retention and cross-pod correlation.
