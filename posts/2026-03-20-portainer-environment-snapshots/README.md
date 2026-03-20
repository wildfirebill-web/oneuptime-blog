# How to Configure Environment Snapshots in Portainer - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Snapshot, Environments, Monitoring, Edge

Description: Configure Portainer environment snapshots to periodically capture the state of your environments for offline viewing and change tracking.

## Introduction

Portainer environment snapshots capture the current state of an environment (containers, images, volumes, services) at a point in time. Snapshots are especially important for Edge environments where connectivity is intermittent. This guide covers configuring snapshot frequency and interpreting snapshot data.

## What Snapshots Contain

A snapshot captures:
- List of running and stopped containers
- Container status, image, ports, and resource usage
- Volumes and their mount points
- Networks and their configurations
- Service configurations (Swarm)
- Node information (Swarm/Kubernetes)

## Configuring Snapshot Interval

### Via Settings UI

1. Go to **Settings** → **Application settings**
2. Find **Snapshot interval**
3. Set the interval (default: `5m`)
4. Save settings

Common formats: `30s`, `5m`, `1h`, `24h`

### Via API

```bash
TOKEN=$(curl -s -X POST \
  https://portainer.example.com/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"adminpassword"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['jwt'])")

# Set snapshot interval to 10 minutes

curl -X PUT \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  https://portainer.example.com/api/settings \
  -d '{
    "SnapshotInterval": "10m"
  }'
```

## Triggering Manual Snapshots

```bash
# Trigger snapshot for all environments
curl -X POST \
  -H "Authorization: Bearer $TOKEN" \
  https://portainer.example.com/api/snapshots

# Trigger snapshot for specific environment
curl -X POST \
  -H "Authorization: Bearer $TOKEN" \
  "https://portainer.example.com/api/endpoints/1/snapshot"
```

## Edge Agent Snapshot Configuration

For async edge environments:

```bash
docker run portainer/agent:latest \
  -e EDGE=1 \
  -e EDGE_ID=unique-id \
  -e EDGE_KEY=portainer-edge-key \
  -e EDGE_ASYNC=1 \
  -e EDGE_PING_INTERVAL=30 \
  -e EDGE_SNAPSHOT_INTERVAL=300
```

## Conclusion

Environment snapshots are a core feature for observability, particularly for edge deployments with intermittent connectivity. Configure the interval based on your needs - more frequent for dynamic production environments, less frequent for stable edge devices.
