# How to Optimize Rancher API Performance

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, API Performance, Kubernetes, Rate Limiting, Pagination, Optimization

Description: Optimize Rancher API performance by implementing pagination, reducing watch connections, tuning API server configuration, and using efficient query patterns.

## Introduction

The Rancher API is a proxy layer over the Kubernetes API. API performance issues manifest as slow CLI commands, delayed automation scripts, and unresponsive cluster management. Understanding how to query efficiently and how Rancher translates requests to Kubernetes API calls is key to resolving performance bottlenecks.

## Step 1: Use Pagination for Large Resource Lists

Listing all pods, events, or deployments without pagination can overwhelm the API server:

```bash
# Bad: Lists all pods in all namespaces at once
kubectl get pods --all-namespaces

# Better: Use --chunk-size to page through results
kubectl get pods --all-namespaces --chunk-size=100

# Best: Filter by namespace and labels
kubectl get pods -n production -l app=myapp
```

```python
# Python kubernetes client - use efficient list_namespaced_pod
from kubernetes import client, config

config.load_kube_config()
v1 = client.CoreV1Api()

# Use field selectors to filter server-side
pods = v1.list_namespaced_pod(
    namespace="production",
    field_selector="status.phase=Running",
    label_selector="app=myapp",
    limit=100,    # Paginate to avoid large responses
)
```

## Step 2: Use Watches Instead of Polling

Polling the API server creates unnecessary load. Use watches for event-driven updates:

```python
# Python watch example - event-driven instead of polling
from kubernetes import client, config, watch

config.load_kube_config()
v1 = client.CoreV1Api()
w = watch.Watch()

# Stream events instead of polling every N seconds
for event in w.stream(v1.list_namespaced_pod, namespace="production"):
    if event['type'] == 'MODIFIED':
        pod = event['object']
        print(f"Pod {pod.metadata.name} changed: {pod.status.phase}")
```

## Step 3: Reduce API Server Load from Rancher

Tune the number of concurrent watches Rancher opens:

```bash
# Check current watch count
kubectl get --raw /metrics | grep apiserver_current_inflight_requests

# Reduce Rancher's watch volume by increasing resync intervals
kubectl set env deployment/rancher -n cattle-system \
  CATTLE_RESYNC_DEFAULT=120 \
  CATTLE_CLUSTER_AGENT_RESYNC=600
```

## Step 4: Configure API Server Rate Limits

Protect the Kubernetes API from being overwhelmed:

```yaml
# kube-apiserver configuration (RKE2)
kube-apiserver-arg:
  - "max-requests-inflight=800"       # Default 400
  - "max-mutating-requests-inflight=400"  # Default 200
  - "request-timeout=60s"
  - "enable-priority-and-fairness=true"   # APF for fair request queuing
```

## Step 5: Cache API Responses in Automation Scripts

```bash
#!/bin/bash
# Cache frequently accessed resources instead of repeated API calls

CACHE_FILE="/tmp/rancher-clusters-cache.json"
CACHE_TTL=300  # 5 minutes

# Only refresh cache if older than TTL
if [[ ! -f "$CACHE_FILE" ]] || \
   [[ $(( $(date +%s) - $(stat -c %Y "$CACHE_FILE") )) -gt $CACHE_TTL ]]; then
  kubectl get clusters.management.cattle.io -o json > "$CACHE_FILE"
fi

# Use the cache
cat "$CACHE_FILE" | jq -r '.items[].metadata.name'
```

## Conclusion

Rancher API performance depends on efficient query patterns, watch usage instead of polling, and appropriate Kubernetes API server configuration. The highest-impact change is adding `--enable-priority-and-fairness` to the API server to ensure fair resource allocation across controllers and user requests.
