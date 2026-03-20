# How to Fix Slow Notification Loading Affecting Bulk Operations

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Performance, Troubleshooting, Notifications

Description: Resolve Portainer performance issues where slow notification loading blocks or delays bulk container operations, stack deployments, and UI interactions.

## Introduction

Portainer's notification system can become a bottleneck in environments with high activity — when you have thousands of events queued up, the notification bell can cause the entire UI to become sluggish. Bulk operations like stopping multiple containers or deploying large stacks can be affected. This guide explains how to manage and optimize notifications.

## Step 1: Check Notification Count

```bash
# Check how many notifications are accumulated
TOKEN=$(curl -s -X POST http://localhost:9000/api/auth \
  -H "Content-Type: application/json" \
  -d '{"Username":"admin","Password":"yourpassword"}' | jq -r .jwt)

curl -s -H "Authorization: Bearer $TOKEN" \
  http://localhost:9000/api/useractivity | \
  jq 'length'
```

If you have thousands of notifications, this is likely causing the slowdown.

## Step 2: Clear All Notifications

```bash
# Clear all notifications via Portainer API
curl -X DELETE \
  -H "Authorization: Bearer $TOKEN" \
  http://localhost:9000/api/notifications

# Or clear via the UI:
# Click the bell icon → "Mark all as read" → "Clear all"
```

## Step 3: Prevent Notification Accumulation

Configure Portainer to limit notification retention:

1. Go to **Settings** → **App Settings**
2. Find **Notification** settings
3. Set a retention period or maximum count

## Step 4: Fix Slow Notification Loading in BoltDB

The notifications are stored in BoltDB. A large number creates slow reads:

```bash
# Stop Portainer and compact the database
docker stop portainer && docker rm portainer

# Run database compaction
docker run --rm \
  -v portainer_data:/data \
  portainer/portainer-ce:latest \
  --compact-db

# Restart
docker run -d \
  -p 9000:9000 \
  -p 9443:9443 \
  --name portainer \
  --restart=unless-stopped \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

## Step 5: Check User Activity Logs

In Portainer Business Edition, activity logs are stored separately and can also accumulate:

```bash
# Check activity log size via API
curl -s -H "Authorization: Bearer $TOKEN" \
  "http://localhost:9000/api/useractivity?limit=1" | \
  jq '.totalCount'

# Clear old activity logs
# Go to: Settings → Authentication → User Activity → Configure retention
```

## Step 6: Identify Which Operations Are Slow

```bash
# Enable debug logging temporarily
docker stop portainer && docker rm portainer
docker run -d \
  -p 9000:9000 \
  --name portainer \
  --restart=unless-stopped \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest \
  --log-level=DEBUG

# Perform the slow operation
# Then check logs for timing info
docker logs portainer 2>&1 | grep -i "notification\|activity\|slow\|ms" | tail -30
```

## Step 7: Workaround — Perform Bulk Operations via CLI

When Portainer UI is slow due to notifications, use CLI for bulk operations:

```bash
# Stop multiple containers from CLI (faster than Portainer UI)
docker stop container1 container2 container3

# Restart all containers in a stack
cd /opt/stacks/mystack
docker compose restart

# Pull and redeploy a stack
docker compose pull && docker compose up -d
```

## Step 8: Use the Portainer API Instead of UI for Bulk Operations

```bash
# Bulk stop containers via API (doesn't wait for UI to load)
TOKEN=$(curl -s -X POST http://localhost:9000/api/auth \
  -H "Content-Type: application/json" \
  -d '{"Username":"admin","Password":"yourpassword"}' | jq -r .jwt)

# Get all running container IDs
CONTAINERS=$(curl -s -H "Authorization: Bearer $TOKEN" \
  "http://localhost:9000/api/endpoints/1/docker/containers/json?filters=%7B%22status%22%3A%5B%22running%22%5D%7D" | \
  jq -r '.[].Id')

# Stop each container
for ID in $CONTAINERS; do
  curl -X POST \
    -H "Authorization: Bearer $TOKEN" \
    "http://localhost:9000/api/endpoints/1/docker/containers/$ID/stop"
  echo "Stopped: $ID"
done
```

## Step 9: Tune the Portainer Database for Performance

```bash
# Move Portainer data to faster storage
# Check current iops
iostat -x 1 3

# If storage is the bottleneck
# Move to an SSD-backed volume
docker stop portainer && docker rm portainer

# Create volume on SSD-backed path
docker volume create --driver local \
  --opt type=none \
  --opt device=/ssd/portainer-data \
  --opt o=bind \
  portainer_ssd_data

# Start with new volume
docker run -d \
  -p 9000:9000 \
  --name portainer \
  --restart=unless-stopped \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_ssd_data:/data \
  portainer/portainer-ce:latest
```

## Step 10: Schedule Notification Cleanup

```bash
#!/bin/bash
# cleanup-portainer-notifications.sh
# Schedule with: 0 3 * * * /opt/scripts/cleanup-portainer-notifications.sh

PORTAINER_URL="http://localhost:9000"
ADMIN_USER="admin"
ADMIN_PASS="yourpassword"

# Get auth token
TOKEN=$(curl -s -X POST "$PORTAINER_URL/api/auth" \
  -H "Content-Type: application/json" \
  -d "{\"Username\":\"$ADMIN_USER\",\"Password\":\"$ADMIN_PASS\"}" | \
  jq -r .jwt)

# Clear notifications
RESULT=$(curl -s -o /dev/null -w "%{http_code}" \
  -X DELETE \
  -H "Authorization: Bearer $TOKEN" \
  "$PORTAINER_URL/api/notifications")

echo "$(date): Notifications cleared. Status: $RESULT"
```

## Conclusion

Notification accumulation is a common but easily overlooked cause of Portainer UI slowness. The immediate fix is to clear all notifications and compact the database. For the long term, schedule nightly cleanup of old notifications, configure retention limits in Portainer settings, and use the API for bulk operations when the UI is sluggish — the API bypasses the notification loading entirely.
