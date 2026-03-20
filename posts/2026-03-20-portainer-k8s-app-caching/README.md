# How to Enable Application Data Caching for Kubernetes in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, Performance, Caching, Configuration

Description: Enable and configure application data caching for Kubernetes environments in Portainer to dramatically reduce API response times and improve UI performance for large clusters.

## Introduction

When managing large Kubernetes clusters in Portainer, the UI can become slow because every page navigation triggers live Kubernetes API calls to list resources. Portainer's application data caching feature stores a snapshot of the Kubernetes cluster state in memory, serving cached responses from Portainer rather than querying the Kubernetes API every time.

## What Application Data Caching Does

When enabled, Portainer:
1. Periodically queries the Kubernetes API and caches the results
2. Serves UI requests from the cache instead of live Kubernetes API calls
3. Provides a configurable refresh interval

**Benefits:**
- Dramatically faster page loads for large clusters
- Reduces load on the Kubernetes API server
- Consistent UI performance regardless of cluster size

**Tradeoff:** Data may be slightly stale (up to the cache interval).

## Step 1: Enable Caching in Portainer UI

1. Log in to Portainer
2. Navigate to **Environments** → select your Kubernetes cluster
3. Click **Edit** (the pencil icon)
4. Scroll to **Kubernetes Settings**
5. Find **Application data caching** toggle
6. Enable it
7. Set the **Cache refresh interval** (default: 60 seconds)
8. Click **Update environment**

## Step 2: Enable via Portainer API

```bash
TOKEN=$(curl -s -X POST http://localhost:9000/api/auth \
  -H "Content-Type: application/json" \
  -d '{"Username":"admin","Password":"yourpassword"}' | jq -r .jwt)

# Get current endpoint configuration

curl -s -H "Authorization: Bearer $TOKEN" \
  http://localhost:9000/api/endpoints/2 | jq '.Kubernetes'

# Update Kubernetes environment to enable caching
curl -X PUT \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  http://localhost:9000/api/endpoints/2 \
  -d '{
    "Name": "my-k8s-cluster",
    "Kubernetes": {
      "Configuration": {
        "UseLoadBalancer": false,
        "UseServerMetrics": false,
        "EnableResourceOverCommit": false,
        "StorageClasses": []
      }
    }
  }'
```

## Step 3: Verify Caching Is Working

```bash
# Measure response time WITHOUT caching
time curl -s -H "Authorization: Bearer $TOKEN" \
  "http://localhost:9000/api/endpoints/2/kubernetes/api/v1/namespaces" -o /dev/null

# Enable caching, wait for first cache population
sleep 65  # Wait for first cache refresh

# Measure response time WITH caching
time curl -s -H "Authorization: Bearer $TOKEN" \
  "http://localhost:9000/api/endpoints/2/kubernetes/api/v1/namespaces" -o /dev/null

# Should be significantly faster with caching
```

## Step 4: Configure Cache Refresh Interval

The refresh interval controls how often the cache is updated:

| Interval | Use Case |
|----------|----------|
| 30 seconds | Dynamic environments with frequent changes |
| 60 seconds (default) | Standard production cluster |
| 120-300 seconds | Stable clusters, reduce API load |
| 600 seconds | Very large clusters, minimal changes |

In Portainer UI: **Environments** → Edit → **Cache refresh interval (seconds)**

## Step 5: Understand Cache Invalidation

```bash
# Cache is invalidated/refreshed when:
# 1. Interval expires (automatic)
# 2. User manually triggers a refresh

# Force cache refresh via API
curl -X POST \
  -H "Authorization: Bearer $TOKEN" \
  http://localhost:9000/api/endpoints/2/kubernetes/cache/refresh

# Or in Portainer UI: click the refresh button on any resource list
```

## Step 6: Monitor Caching Performance

```bash
# Check Portainer logs for cache activity
docker logs portainer 2>&1 | grep -i "cache\|kubernetes\|refresh" | tail -20

# Monitor Portainer's memory usage (cache lives in memory)
docker stats portainer --no-stream

# Large clusters can require more memory for caching
# Consider increasing Portainer's memory limit
docker run -d \
  --memory="1g" \
  --name portainer \
  [other options] \
  portainer/portainer-ce:latest
```

## Step 7: Caching for Multiple Kubernetes Environments

Each Kubernetes environment has its own cache. Configure appropriately per cluster size:

```bash
# Get all Kubernetes environments
curl -s -H "Authorization: Bearer $TOKEN" \
  http://localhost:9000/api/endpoints | \
  jq '.[] | select(.Type == 3) | {id: .Id, name: .Name}'

# Enable caching for each Kubernetes endpoint separately
# via Portainer UI: Environments → Edit for each cluster
```

## Step 8: What Gets Cached

The following Kubernetes resources are cached when enabled:
- Namespaces
- Pods
- Services
- Deployments, StatefulSets, DaemonSets
- ConfigMaps and Secrets (names only)
- Persistent Volumes and Claims
- Ingresses
- Node information

## Step 9: Disable Caching for Troubleshooting

If you suspect the cache is serving stale data:

1. **Portainer UI**: Environments → Edit → toggle caching off → Save
2. **API**: Set `Kubernetes.Configuration.UseCache` to false
3. **Workaround**: Click the refresh button to force a cache bust

## Conclusion

Application data caching for Kubernetes is one of the most impactful performance features in Portainer for large clusters. Enable it for any Kubernetes environment with more than 50 namespaces or 200+ resources. The default 60-second interval works well for most deployments - increase it to 300+ seconds for stable production clusters to minimize Kubernetes API server load.
