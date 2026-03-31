# How to Use Watchtower Monitor-Only Mode with Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Watchtower, Monitor, Notification, Update

Description: Learn how to run Watchtower in monitor-only mode alongside Portainer to receive update notifications without automatic container restarts, giving you visibility into available updates while...

## Introduction

Watchtower's monitor-only mode checks for new container image versions and sends notifications, but does not automatically pull new images or restart containers. This provides the visibility benefits of Watchtower - knowing when updates are available - without the risk of unexpected container restarts in production. This guide covers deploying and using Watchtower in monitor-only mode.

## Prerequisites

- Portainer deployed for container management
- At least one notification channel configured (Slack, email, etc.)
- Docker socket accessible to Watchtower

## Step 1: Deploy Watchtower in Monitor-Only Mode

```yaml
# Portainer stack - monitor-only Watchtower

version: "3.8"

services:
  watchtower:
    image: containrrr/watchtower:latest
    container_name: watchtower-monitor
    restart: unless-stopped
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      # Core: enable monitor-only mode
      WATCHTOWER_MONITOR_ONLY: "true"         # Check for updates but DO NOT apply them

      # Poll every 6 hours for update availability
      WATCHTOWER_POLL_INTERVAL: "21600"

      # Notifications (required to get value from monitor-only mode)
      WATCHTOWER_NOTIFICATIONS: "slack"
      WATCHTOWER_NOTIFICATION_SLACK_HOOK_URL: "${SLACK_WEBHOOK_URL}"
      WATCHTOWER_NOTIFICATION_SLACK_IDENTIFIER: "Update Monitor@production"
      WATCHTOWER_NOTIFICATION_SLACK_CHANNEL: "#update-alerts"

      # Report level: info shows updates when found
      WATCHTOWER_NOTIFICATIONS_LEVEL: "info"
```

## Step 2: Per-Container Monitor-Only Overrides

Even with Watchtower set to auto-update globally, mark individual containers as monitor-only:

```yaml
# Portainer application stack
services:
  # This container: auto-update (default behavior)
  nginx:
    image: nginx:alpine
    labels:
      - "com.centurylinklabs.watchtower.enable=true"

  # This container: monitor only (override global setting)
  postgres:
    image: postgres:15-alpine
    labels:
      - "com.centurylinklabs.watchtower.enable=true"
      - "com.centurylinklabs.watchtower.monitor-only=true"    # Watch but don't update
```

## Step 3: Monitor-Only with Manual Update Workflow

Monitor-only mode pairs well with a manual update workflow via Portainer:

```bash
#!/bin/bash
# review-and-update.sh - Run weekly to apply pending updates

# Get notification of outdated containers (from Watchtower monitor)
# Then for each outdated container, update manually:

CONTAINER_NAME="${1}"

# 1. Pull the new image
docker pull $(docker inspect "$CONTAINER_NAME" | jq -r '.[].Config.Image')

# 2. Get the current container config
IMAGE=$(docker inspect "$CONTAINER_NAME" | jq -r '.[].Config.Image')
ENV=$(docker inspect "$CONTAINER_NAME" | jq -r '.[].Config.Env[]' | awk '{print "-e " $0}' | tr '\n' ' ')

# 3. Stop and remove the old container
docker stop "$CONTAINER_NAME"
docker rm "$CONTAINER_NAME"

# 4. Start with new image (preserving volumes via docker-compose is safer)
echo "Update $CONTAINER_NAME to new image: $IMAGE"
echo "Use: docker compose pull && docker compose up -d in the stack directory"
```

In Portainer, the manual update workflow is:
1. **Stacks** → select the stack with outdated containers
2. Click **Pull and redeploy**
3. Or edit the stack, change the image tag, and click **Update the stack**

## Step 4: Monitor Specific Containers Only

Watch a subset of containers while ignoring others:

```yaml
services:
  watchtower:
    image: containrrr/watchtower:latest
    environment:
      WATCHTOWER_MONITOR_ONLY: "true"
      WATCHTOWER_POLL_INTERVAL: "21600"
      WATCHTOWER_NOTIFICATIONS: "slack"
      WATCHTOWER_NOTIFICATION_SLACK_HOOK_URL: "${SLACK_WEBHOOK_URL}"
    # Specify container names to monitor (positional arguments)
    # Monitors ONLY these containers:
    command: >
      --monitor-only
      --interval 21600
      nginx
      myapp-api
      myapp-frontend
```

## Step 5: Graduated Update Strategy

Use monitor-only on production, auto-update on staging:

```yaml
# Production server: monitor only
services:
  watchtower:
    image: containrrr/watchtower:latest
    environment:
      WATCHTOWER_MONITOR_ONLY: "true"            # Production: notify only
      WATCHTOWER_NOTIFICATIONS: "slack"
      WATCHTOWER_NOTIFICATION_SLACK_HOOK_URL: "${SLACK_WEBHOOK_URL}"
      WATCHTOWER_NOTIFICATION_SLACK_IDENTIFIER: "PRODUCTION Monitor"

---
# Staging server: auto-update to validate new images
services:
  watchtower:
    image: containrrr/watchtower:latest
    environment:
      WATCHTOWER_MONITOR_ONLY: "false"           # Staging: auto-update
      WATCHTOWER_POLL_INTERVAL: "3600"           # Check hourly on staging
      WATCHTOWER_NOTIFICATIONS: "slack"
      WATCHTOWER_NOTIFICATION_SLACK_HOOK_URL: "${SLACK_WEBHOOK_URL}"
      WATCHTOWER_NOTIFICATION_SLACK_IDENTIFIER: "STAGING Auto-Updater"
```

## Step 6: Combine with Portainer Webhooks for Semi-Automation

Use monitor-only notification to trigger a human-approved deployment:

```bash
# When Watchtower notifies about an update via Slack:
# 1. Team reviews the update (release notes, security advisories)
# 2. Approved team member triggers Portainer webhook manually

# Portainer stack webhook for manual trigger:
curl -X POST "https://portainer.example.com/api/stacks/webhooks/YOUR-WEBHOOK-UUID"

# Or via Portainer API with auth:
curl -X POST -H "Authorization: Bearer $TOKEN" \
  "https://portainer.example.com/api/stacks/1/git/redeploy" \
  -d '{"pullImage": true}'
```

## Step 7: Check Watchtower Monitor Logs

```bash
# View monitor activity
docker logs watchtower-monitor --follow

# See what Watchtower found (debug level shows more details)
docker logs watchtower-monitor 2>&1 | grep -i "found\|outdated\|update\|monitor"

# Sample output in monitor-only mode:
# INF Checking container: nginx, image: nginx:alpine
# INF Found newer image for container: nginx
# INF Session done - Checked 8 containers, found 2 with new images
# (No restarts because monitor-only is enabled)
```

## Conclusion

Monitor-only mode gives you the update visibility of Watchtower without the operational risk of automatic container restarts in production. Pair it with Slack notifications so your team sees update alerts in real time, and use Portainer's stack management to apply updates manually after reviewing release notes. For a graduated approach, run auto-update on staging and monitor-only on production - if an image update works smoothly on staging, you have higher confidence when applying it to production through Portainer.
