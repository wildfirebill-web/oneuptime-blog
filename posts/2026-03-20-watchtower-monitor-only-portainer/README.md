# How to Use Watchtower in Monitor-Only Mode with Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Watchtower, Docker, Container Updates, Monitoring, DevOps

Description: Learn how to run Watchtower in monitor-only mode alongside Portainer to get notifications about available container image updates without automatically applying them.

## Introduction

Watchtower is a Docker container that watches running containers and updates them when new images are available. In monitor-only mode, Watchtower checks for updates and sends notifications without actually updating containers, giving you visibility and control over when updates are applied.

## Why Monitor-Only Mode?

Monitor-only mode is ideal when:
- You want notifications about available updates
- Auto-updating containers is too risky for your environment
- You use Portainer to manually control update timing
- You need change management approval before updates

## Docker Compose Configuration

Add Watchtower alongside Portainer in your stack:

```yaml
version: "3.8"

services:
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    restart: unless-stopped
    ports:
      - "9000:9000"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data

  watchtower:
    image: containrrr/watchtower:latest
    container_name: watchtower
    restart: unless-stopped
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      # Monitor only - do not update
      - WATCHTOWER_MONITOR_ONLY=true
      # Check every 24 hours
      - WATCHTOWER_SCHEDULE=0 0 0 * * *
      # Notification settings
      - WATCHTOWER_NOTIFICATION_EMAIL_FROM=watchtower@example.com
      - WATCHTOWER_NOTIFICATION_EMAIL_TO=ops@example.com
      - WATCHTOWER_NOTIFICATION_EMAIL_SERVER=smtp.example.com
      - WATCHTOWER_NOTIFICATION_EMAIL_SERVER_PORT=587
      - WATCHTOWER_NOTIFICATION_EMAIL_SERVER_USER=watchtower
      - WATCHTOWER_NOTIFICATION_EMAIL_SERVER_PASSWORD=${SMTP_PASSWORD}
      - WATCHTOWER_NOTIFICATIONS=email

volumes:
  portainer_data:
```

## Slack Notifications

```yaml
environment:
  - WATCHTOWER_MONITOR_ONLY=true
  - WATCHTOWER_NOTIFICATIONS=slack
  - WATCHTOWER_NOTIFICATION_SLACK_HOOK_URL=https://hooks.slack.com/services/xxx/yyy/zzz
  - WATCHTOWER_NOTIFICATION_SLACK_IDENTIFIER=watchtower-prod
```

## Running a Manual Check

```bash
docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  containrrr/watchtower \
  --monitor-only \
  --run-once \
  --notifications=slack \
  --notification-slack-hook-url="https://hooks.slack.com/..."
```

## Including/Excluding Specific Containers

Monitor only labeled containers:

```yaml
environment:
  - WATCHTOWER_MONITOR_ONLY=true
  - WATCHTOWER_LABEL_ENABLE=true
```

Then label containers to monitor:

```yaml
labels:
  - "com.centurylinklabs.watchtower.enable=true"
```

## Checking Watchtower Logs

```bash
docker logs watchtower --tail=50 -f
```

## Conclusion

Running Watchtower in monitor-only mode alongside Portainer provides a useful update awareness layer without the risk of unexpected automatic updates. Notifications keep your team informed about available updates, and Portainer provides the interface to apply them on your schedule.
