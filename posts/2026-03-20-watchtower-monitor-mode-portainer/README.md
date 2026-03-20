# How to Use Watchtower Monitor-Only Mode with Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Watchtower, Monitor Mode, Container Updates, Notifications

Description: Learn how to run Watchtower in monitor-only mode alongside Portainer to get notified when container image updates are available without automatically applying them.

## What Is Monitor-Only Mode?

Monitor-only mode tells Watchtower to check for new images and notify you when updates are available — but not to apply them. You decide when to update, and Portainer's image update feature applies them.

This is ideal for:
- Production environments where changes need approval
- Services where update timing matters
- Teams that want awareness but manual control

## Deploy Watchtower in Monitor-Only Mode

```yaml
# In Portainer: Stacks > Add Stack > watchtower-monitor
version: "3.8"

services:
  watchtower:
    image: containrrr/watchtower:latest
    restart: unless-stopped
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      # Monitor only — never update
      - WATCHTOWER_MONITOR_ONLY=true
      # Check every 6 hours
      - WATCHTOWER_POLL_INTERVAL=21600
      # Notify via Slack when updates are available
      - WATCHTOWER_NOTIFICATIONS=slack
      - WATCHTOWER_NOTIFICATION_SLACK_HOOK_URL=${SLACK_WEBHOOK_URL}
      - WATCHTOWER_NOTIFICATION_SLACK_IDENTIFIER=watchtower-monitor
      - WATCHTOWER_NOTIFICATIONS_LEVEL=info
```

## Monitor-Only for All Containers Except One

You can set monitor-only globally but allow specific containers to auto-update:

```yaml
environment:
  - WATCHTOWER_MONITOR_ONLY=true    # Default: monitor only
```

Override per container with:

```yaml
labels:
  # This container ignores global monitor-only and auto-updates
  - "com.centurylinklabs.watchtower.monitor-only=false"
```

## Per-Container Monitor-Only

Override the global setting per container:

```yaml
services:
  critical-app:
    image: myapp:latest
    labels:
      # Monitor only for this container regardless of global setting
      - "com.centurylinklabs.watchtower.monitor-only=true"

  non-critical-tool:
    image: mytool:latest
    labels:
      # Auto-update for this container
      - "com.centurylinklabs.watchtower.monitor-only=false"
      - "com.centurylinklabs.watchtower.enable=true"
```

## Workflow: Monitor → Review → Update via Portainer

1. Watchtower detects a new image and sends a Slack notification
2. Review the changelog for the updated image
3. In Portainer: **Containers → select container → Recreate** (check "Re-pull image")
4. Or update via stack: **Stacks → your-stack → Editor → Update the Stack**

## Testing Monitor Mode

Run once and check output:

```bash
docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  containrrr/watchtower \
  --monitor-only --run-once --debug 2>&1 | grep -E "Found|update|monitor"
```

Output example:

```
level=debug msg="Found new registry image for myapp/frontend:latest"
level=info msg="Skipping container update (monitor only): frontend"
```

## Notification Comparison

| Mode | Notification Content |
|------|---------------------|
| Update mode | "Updated containers: webapp (v1 → v2)" |
| Monitor mode | "Updates available: webapp (v1 → v2) — not applied" |

## Combining with Portainer Webhooks

A cleaner workflow uses Portainer's webhook + CI/CD:

1. Watchtower monitors and notifies on new images
2. A human or bot reviews and triggers a Portainer webhook to redeploy
3. Portainer redeploys the stack with the new image

This gives you awareness from Watchtower and control through Portainer.

## Conclusion

Watchtower in monitor-only mode bridges the gap between fully manual and fully automated updates. Your team gets proactive alerts when images need updating, but updates only happen when deliberately triggered through Portainer — preserving the change control process while eliminating the need to manually check for new images.
