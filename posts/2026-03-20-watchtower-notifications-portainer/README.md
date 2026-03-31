# How to Set Up Watchtower Notifications with Portainer - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Watchtower, Notification, Slack, Email

Description: Learn how to configure Watchtower to send notifications when containers are updated, failed, or skipped, using Slack, email, and other notification services with your Portainer deployment.

## Notification Options

Watchtower supports multiple notification channels:

| Channel | Config Variable Prefix |
|---------|----------------------|
| Slack | `WATCHTOWER_NOTIFICATION_SLACK_*` |
| Email (SMTP) | `WATCHTOWER_NOTIFICATION_EMAIL_*` |
| Microsoft Teams | `WATCHTOWER_NOTIFICATION_MSTEAMS_*` |
| Gotify | `WATCHTOWER_NOTIFICATION_GOTIFY_*` |
| Shoutrrr (generic) | `WATCHTOWER_NOTIFICATION_URL` |

## Slack Notifications

```yaml
# In Portainer: Stacks > Add Stack > watchtower

services:
  watchtower:
    image: containrrr/watchtower:latest
    restart: unless-stopped
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - WATCHTOWER_CLEANUP=true
      - WATCHTOWER_LABEL_ENABLE=true
      - WATCHTOWER_POLL_INTERVAL=3600
      # Slack settings
      - WATCHTOWER_NOTIFICATIONS=slack
      - WATCHTOWER_NOTIFICATION_SLACK_HOOK_URL=${SLACK_WEBHOOK_URL}
      - WATCHTOWER_NOTIFICATION_SLACK_IDENTIFIER=watchtower-prod
      - WATCHTOWER_NOTIFICATION_SLACK_CHANNEL=#deployments
      - WATCHTOWER_NOTIFICATION_SLACK_ICON_EMOJI=:whale:
```

Create a Slack incoming webhook at `api.slack.com/messaging/webhooks` and add the URL as `SLACK_WEBHOOK_URL` in Portainer's environment variables.

## Email Notifications via SMTP

```yaml
environment:
  - WATCHTOWER_NOTIFICATIONS=email
  - WATCHTOWER_NOTIFICATION_EMAIL_FROM=watchtower@yourdomain.com
  - WATCHTOWER_NOTIFICATION_EMAIL_TO=team@yourdomain.com
  - WATCHTOWER_NOTIFICATION_EMAIL_SERVER=smtp.gmail.com
  - WATCHTOWER_NOTIFICATION_EMAIL_SERVER_PORT=587
  - WATCHTOWER_NOTIFICATION_EMAIL_SERVER_USER=watchtower@yourdomain.com
  - WATCHTOWER_NOTIFICATION_EMAIL_SERVER_PASSWORD=${SMTP_PASSWORD}
  - WATCHTOWER_NOTIFICATION_EMAIL_SUBJECT_TAG=[prod-server]
  - WATCHTOWER_NOTIFICATION_EMAIL_DELAY=2    # Batch notifications, 2 second delay
```

## Microsoft Teams Notifications

```yaml
environment:
  - WATCHTOWER_NOTIFICATIONS=msteams
  - WATCHTOWER_NOTIFICATION_MSTEAMS_HOOK_URL=${TEAMS_WEBHOOK_URL}
  - WATCHTOWER_NOTIFICATION_MSTEAMS_USE_LOG_DATA=true
```

## Generic Notifications via Shoutrrr

Shoutrrr supports many services via a unified URL format:

```yaml
environment:
  # Discord
  - WATCHTOWER_NOTIFICATION_URL=discord://token@webhookid

  # Telegram
  - WATCHTOWER_NOTIFICATION_URL=telegram://token@telegram?chats=@channelname

  # Pushover
  - WATCHTOWER_NOTIFICATION_URL=pushover://shoutrrr:apiToken@userKey

  # Multiple services
  - WATCHTOWER_NOTIFICATION_URL=slack://token@channel discord://token@webhookid
```

## Controlling Notification Level

```yaml
environment:
  # Only notify on updates (not on "no update available")
  - WATCHTOWER_NOTIFICATIONS_LEVEL=info

  # Available levels: panic, fatal, error, warn, info, debug, trace
  # 'warn' = only errors/warnings (quiet mode)
  # 'info' = updates + errors (recommended)
  # 'debug' = everything (verbose)
```

## Testing Notifications

Run Watchtower once with debug to verify notifications work:

```bash
docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -e WATCHTOWER_NOTIFICATIONS=slack \
  -e WATCHTOWER_NOTIFICATION_SLACK_HOOK_URL=https://hooks.slack.com/... \
  -e WATCHTOWER_NOTIFICATION_SLACK_IDENTIFIER=test \
  containrrr/watchtower \
  --run-once --monitor-only --debug
```

## Sample Notification Message

A Slack notification from Watchtower looks like:

```text
🐳 watchtower-prod

Updated containers:
• myapp: myorg/myapp:latest (sha256:abc123 → sha256:def456)
• frontend: myorg/frontend:latest (sha256:111 → sha256:222)

Skipped containers (no update available):
• nginx: nginx:alpine

Completed at: 2026-03-20 03:00:01 UTC
```

## Conclusion

Watchtower notifications keep your team informed when auto-updates occur, providing an audit trail of what changed and when. Using Portainer's environment variable management, you can securely store webhook URLs and SMTP credentials without hardcoding them in your stack definition.
