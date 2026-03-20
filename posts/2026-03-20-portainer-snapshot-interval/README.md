# How to Configure the Snapshot Interval in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Configuration, Snapshot, Performance, Monitoring

Description: A guide to configuring Portainer's environment snapshot interval to balance freshness with performance overhead.

## Overview

Portainer polls connected environments (Docker hosts, Kubernetes clusters) at a regular interval to capture their state (running containers, images, volumes, etc.) - called a "snapshot". The default interval is 60 seconds. Adjusting this interval lets you balance data freshness against the overhead of polling many environments.

## Prerequisites

- Portainer CE or Business Edition
- Admin access to Portainer

## Understanding Snapshots

Portainer snapshots capture:
- Running/stopped containers and their states
- Images pulled on the host
- Networks and volumes
- Stack status
- Resource usage statistics

These snapshots are what Portainer displays in its dashboards.

## Method 1: Configure via UI

1. Navigate to **Settings** → **App Settings**
2. Find the **Snapshot interval** field
3. Set the desired interval in seconds (default: 60)
4. Click **Save settings**

Common values:
- `30` - High-frequency polling (heavier load)
- `60` - Default, good for most environments
- `300` - Low-frequency for many environments or limited resources
- `3600` - Hourly, for static or less dynamic environments

## Method 2: Configure via API

```bash
PORTAINER_URL="https://portainer.example.com:9443"
TOKEN="your-admin-token"

# Set snapshot interval to 120 seconds

curl -X PUT \
  "${PORTAINER_URL}/api/settings" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"SnapshotInterval": "120"}'

# Verify
curl -s -H "Authorization: Bearer ${TOKEN}" \
  "${PORTAINER_URL}/api/settings" \
  | jq '.SnapshotInterval'
```

## Method 3: Set at Startup via Flag

```bash
docker run -d \
  -p 9443:9443 \
  -p 8000:8000 \
  --name portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest \
  --snapshot-interval 120
```

## Performance Considerations

| Environments | Recommended Interval | Notes |
|---|---|---|
| 1-5 | 30-60s | Default is fine |
| 5-20 | 60-120s | Slightly increased to reduce load |
| 20-50 | 120-300s | Balance freshness vs overhead |
| 50+ | 300-600s | High interval to prevent overload |
| Edge (remote) | 300s+ | Network latency makes frequent polls expensive |

## Impact of Snapshot Interval on Portainer

```bash
# High-frequency snapshotting effects:
# - More accurate real-time data in dashboard
# - Higher CPU/memory on Portainer server
# - More API calls to Docker/Kubernetes endpoints
# - Higher network traffic to remote environments

# Low-frequency snapshotting effects:
# - Dashboard may show stale container states
# - Lower resource consumption
# - Better for environments with many endpoints
```

## Triggering Manual Snapshot

Portainer 2.x supports triggering a manual snapshot refresh:

```bash
# Force snapshot of a specific endpoint
ENDPOINT_ID=1
curl -X POST \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/snapshot" \
  -H "Authorization: Bearer ${TOKEN}"
```

## Monitoring Snapshot Performance

```bash
# Check Portainer logs for snapshot timing
docker logs portainer 2>&1 | grep -i "snapshot"

# Monitor Portainer memory usage
docker stats portainer --no-stream --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}"
```

## Conclusion

The snapshot interval is a tuning knob that directly affects the trade-off between data freshness and system load. For small deployments (1-5 environments), the default 60-second interval is ideal. For larger deployments with many remote endpoints, increasing the interval to 120-300 seconds significantly reduces load while still providing reasonably up-to-date information in the Portainer dashboard.
