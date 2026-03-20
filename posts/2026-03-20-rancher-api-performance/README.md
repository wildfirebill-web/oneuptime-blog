# How to Optimize Rancher API Performance - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, API, Performance, Optimization, Kubernetes

Description: Optimize Rancher's API performance through connection pooling, caching, pagination, and API query optimization for faster management operations.

## Introduction

Rancher's API is the backbone of its management capabilities-used by the UI, CLI, Terraform provider, and automation scripts. Poor API performance manifests as slow UI, timing out kubectl operations, and failed automation. This guide covers diagnosing and optimizing Rancher API performance.

## Prerequisites

- Running Rancher installation
- Prometheus/Grafana monitoring stack
- kubectl and curl access

## Step 1: Diagnose API Performance Issues

```bash
# Measure API response times

time curl -sk \
  -H "Authorization: Bearer $TOKEN" \
  "https://rancher.example.com/v3/clusters" \
  | jq '.data | length'

# Check API server audit logs for slow requests
kubectl logs -n cattle-system deployment/rancher \
  | grep -E '"latency":[0-9]{4,}' \
  | jq '{uri: .requestURI, latency: .latency}' \
  | sort -t: -k2 -rn | head -20

# Check active websocket connections
kubectl exec -n cattle-system \
  $(kubectl get pod -n cattle-system -l app=rancher -o name | head -1) \
  -- netstat -an | grep ESTABLISHED | wc -l
```

## Step 2: Enable Rancher API Caching

```bash
# Rancher caches some resources in memory
# Increase Norman cache size via environment variables

# Edit Rancher deployment
kubectl edit deployment rancher -n cattle-system

# Add these environment variables to increase cache sizes
env:
  - name: CATTLE_SERVER_URL
    value: "https://rancher.example.com"
  # Increase Steve (API v3) cache
  - name: CATTLE_STEVE_CACHE_SIZE
    value: "500"
  # Increase Norman cache
  - name: CATTLE_NORMAN_CACHE_SIZE
    value: "500"
```

## Step 3: Use API Pagination Effectively

```bash
# Avoid fetching all resources at once
# Use pagination for large result sets

# Bad: fetch all pods across all clusters
curl -H "Authorization: Bearer $TOKEN" \
  "https://rancher.example.com/v3/pods"

# Good: use pagination
PAGE_SIZE=100
MARKER=""

while true; do
  RESPONSE=$(curl -s \
    -H "Authorization: Bearer $TOKEN" \
    "https://rancher.example.com/v3/pods?limit=$PAGE_SIZE&marker=$MARKER")

  echo $RESPONSE | jq '.data[].metadata.name'

  MARKER=$(echo $RESPONSE | jq -r '.pagination.next // empty' | grep -oP 'marker=\K[^&]+')
  [ -z "$MARKER" ] && break
done
```

## Step 4: Filter API Requests

```bash
# Use label selectors and field selectors to reduce response size

# Filter by namespace
curl -H "Authorization: Bearer $TOKEN" \
  "https://rancher.example.com/v3/namespaces?projectId=c-xxxxx:p-xxxxx"

# Filter by labels
curl -H "Authorization: Bearer $TOKEN" \
  "https://rancher.example.com/k8s/clusters/c-xxxxx/api/v1/pods?\
labelSelector=app=myapp&fieldSelector=status.phase=Running"

# Use sparse fieldsets to reduce payload
curl -H "Authorization: Bearer $TOKEN" \
  "https://rancher.example.com/v3/clusters" | \
  jq '[.data[] | {id, name, state: .state}]'
```

## Step 5: Configure Connection Keep-Alive

```python
# Python client with connection pooling
import requests
from requests.adapters import HTTPAdapter

session = requests.Session()
adapter = HTTPAdapter(
    pool_connections=10,
    pool_maxsize=20,
    max_retries=3
)
session.mount('https://', adapter)
session.headers.update({'Authorization': f'Bearer {token}'})

# Reuse the session for multiple requests
# This avoids TLS handshake overhead
clusters = session.get('https://rancher.example.com/v3/clusters').json()
```

## Step 6: Use Watch for Real-Time Updates

```bash
# Instead of polling, use watch for real-time updates
# This reduces API load significantly

# Watch cluster state changes
curl -N -H "Authorization: Bearer $TOKEN" \
  "https://rancher.example.com/v3/clusters?watch=true" | \
  while read line; do
    echo $line | jq -r 'select(.type == "changed") | "\(.data.name): \(.data.state)"'
  done

# Use kubectl watch for managed clusters
kubectl get pods -n production -w --context=rancher-production
```

## Step 7: Implement Local Caching in Automation

```python
# Cache API responses locally with TTL
import time
from functools import lru_cache
import requests

CACHE_TTL = 60  # 60 seconds

class RancherClient:
    def __init__(self, url, token):
        self.url = url
        self.session = requests.Session()
        self.session.headers['Authorization'] = f'Bearer {token}'
        self._cluster_cache = {}
        self._cache_time = {}

    def get_clusters(self):
        cache_key = 'clusters'
        now = time.time()

        if cache_key in self._cluster_cache and \
           now - self._cache_time.get(cache_key, 0) < CACHE_TTL:
            return self._cluster_cache[cache_key]

        response = self.session.get(f'{self.url}/v3/clusters')
        self._cluster_cache[cache_key] = response.json()['data']
        self._cache_time[cache_key] = now
        return self._cluster_cache[cache_key]
```

## Step 8: Monitor API Metrics

```yaml
# rancher-api-alerts.yaml - Prometheus alerts for API issues
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: rancher-api-alerts
  namespace: cattle-monitoring-system
spec:
  groups:
    - name: rancher-api
      rules:
        - alert: RancherAPIHighLatency
          expr: |
            histogram_quantile(0.99,
              rate(rancher_api_request_duration_seconds_bucket[5m])
            ) > 2
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Rancher API p99 latency > 2 seconds"

        - alert: RancherAPIHighErrorRate
          expr: |
            rate(rancher_api_request_total{code=~"5.."}[5m]) /
            rate(rancher_api_request_total[5m]) > 0.05
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "Rancher API error rate > 5%"
```

## Conclusion

Rancher API performance optimization is an iterative process. Start by identifying slow endpoints through audit log analysis, then apply targeted fixes: increase cache sizes, use proper pagination and filtering, implement connection pooling in clients, and leverage watch instead of polling for real-time monitoring. For automation and Terraform workflows, adding local client-side caching for rarely-changing resources like cluster metadata can significantly reduce API load and improve overall management performance.
