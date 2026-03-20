# How to Set Up Watchtower Notifications with Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Watchtower, Notifications, Slack, Email

Description: Learn how to configure Watchtower to send notifications when containers are updated, including Slack, email, Microsoft Teams, and generic webhook integrations for Portainer-managed environments.

## Introduction

Watchtower notifications keep your team informed when containers are automatically updated. Without notifications, auto-updates happen silently - you won't know when a new image was deployed or if an update caused issues. This guide covers configuring Watchtower notifications for common platforms when running as a Portainer stack.

## Prerequisites

- Watchtower deployed as a Portainer stack
- Access to your notification platform (Slack, email server, Teams, etc.)
- Webhook URLs or SMTP credentials ready

## Step 1: Slack Notifications

```yaml
# Portainer stack - Watchtower with Slack notifications

services:
  watchtower:
    image: containrrr/watchtower:latest
    container_name: watchtower
    restart: unless-stopped
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      WATCHTOWER_POLL_INTERVAL: "86400"
      WATCHTOWER_CLEANUP: "true"

      # Slack configuration
      WATCHTOWER_NOTIFICATIONS: "slack"
      WATCHTOWER_NOTIFICATION_SLACK_HOOK_URL: "${SLACK_WEBHOOK_URL}"
      WATCHTOWER_NOTIFICATION_SLACK_IDENTIFIER: "Watchtower@production-server"
      WATCHTOWER_NOTIFICATION_SLACK_CHANNEL: "#container-updates"
      WATCHTOWER_NOTIFICATION_SLACK_ICON_EMOJI: ":whale:"

      # Only notify on successful updates (not on scan with no updates)
      WATCHTOWER_NOTIFICATIONS_LEVEL: "info"    # debug, info, warn, error, fatal
```

Set the Portainer environment variable `SLACK_WEBHOOK_URL` to your Slack incoming webhook URL.

## Step 2: Email Notifications

```yaml
services:
  watchtower:
    image: containrrr/watchtower:latest
    environment:
      WATCHTOWER_POLL_INTERVAL: "86400"
      WATCHTOWER_CLEANUP: "true"

      # Email (SMTP) configuration
      WATCHTOWER_NOTIFICATIONS: "email"
      WATCHTOWER_NOTIFICATION_EMAIL_FROM: "watchtower@example.com"
      WATCHTOWER_NOTIFICATION_EMAIL_TO: "devops@example.com"
      WATCHTOWER_NOTIFICATION_EMAIL_SERVER: "smtp.gmail.com"
      WATCHTOWER_NOTIFICATION_EMAIL_SERVER_PORT: "587"
      WATCHTOWER_NOTIFICATION_EMAIL_SERVER_USER: "watchtower@gmail.com"
      WATCHTOWER_NOTIFICATION_EMAIL_SERVER_PASSWORD: "${SMTP_PASSWORD}"
      WATCHTOWER_NOTIFICATION_EMAIL_SUBJECT_TAG: "[production]"    # Prefix for subject line
      WATCHTOWER_NOTIFICATION_EMAIL_DELAY: "2"    # Seconds between checks before sending email
```

## Step 3: Microsoft Teams Notifications

```yaml
services:
  watchtower:
    environment:
      WATCHTOWER_NOTIFICATIONS: "msteams"
      WATCHTOWER_NOTIFICATION_MSTEAMS_HOOK_URL: "${TEAMS_WEBHOOK_URL}"
      WATCHTOWER_NOTIFICATION_MSTEAMS_USE_LOG_DATA: "true"    # Include log details in message
```

Create the Teams webhook:
1. In Teams, go to your channel → **...** → **Connectors**
2. Find **Incoming Webhook** → Configure
3. Copy the webhook URL

## Step 4: Generic Webhook (for Custom Integrations)

Send notifications to any HTTP endpoint:

```yaml
services:
  watchtower:
    environment:
      WATCHTOWER_NOTIFICATIONS: "gotify"    # or use shoutrrr for generic webhooks
      # For generic webhooks, use shoutrrr URL format:
      WATCHTOWER_NOTIFICATION_URL: "generic+https://webhook.example.com/watchtower?template=json&headers=Authorization=Bearer+TOKEN"
```

## Step 5: Shoutrrr URL Format (Multi-Provider)

Watchtower uses the Shoutrrr library for notifications, supporting a URL-based format:

```yaml
services:
  watchtower:
    environment:
      # Multiple notification channels using WATCHTOWER_NOTIFICATION_URL
      # Slack via shoutrrr
      WATCHTOWER_NOTIFICATION_URL: "slack://WORKSPACE/CHANNEL/WEBHOOK_TOKEN"

      # For multiple notifications, use comma separation (some versions)
      # Or set WATCHTOWER_NOTIFICATION_URL multiple times with numbered suffixes
```

```bash
# Shoutrrr URL examples:
# Slack:     slack://TOKEN:WEBHOOK@CHANNEL
# Discord:   discord://TOKEN@CHANNELID
# Telegram:  telegram://TOKEN@telegram?channels=CHATID
# Email:     smtp://USER:PASS@HOST:PORT/?from=FROM&to=TO
# Gotify:    gotify://HOSTNAME/TOKEN
# Pushover:  pushover://TOKEN@USER/?devices=DEVICE
```

## Step 6: Notification Level Control

Control the verbosity of notifications:

```yaml
services:
  watchtower:
    environment:
      # Send notification even when no updates found (debug)
      WATCHTOWER_NOTIFICATIONS_LEVEL: "debug"

      # Only notify on actual updates or errors (info - recommended)
      WATCHTOWER_NOTIFICATIONS_LEVEL: "info"

      # Only notify on warnings and errors (warn)
      WATCHTOWER_NOTIFICATIONS_LEVEL: "warn"

      # Report containers that were updated (vs all containers checked)
      WATCHTOWER_INCLUDE_RESTARTING: "true"    # Include containers being restarted
```

## Step 7: Test Notifications Before Deployment

```bash
# Test that notifications work by running Watchtower once with --debug
docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -e WATCHTOWER_NOTIFICATIONS=slack \
  -e WATCHTOWER_NOTIFICATION_SLACK_HOOK_URL="https://hooks.slack.com/services/YOUR/WEBHOOK" \
  -e WATCHTOWER_NOTIFICATION_SLACK_IDENTIFIER="Test" \
  -e WATCHTOWER_NOTIFICATIONS_LEVEL=debug \
  containrrr/watchtower \
  --run-once \
  --debug

# You should receive a test notification in Slack immediately
```

## Step 8: Sample Notification Message

A typical Watchtower Slack notification looks like:

```text
Watchtower@production-server: Updated containers

Updated:
- /nginx (containrrr/nginx:alpine → containrrr/nginx:alpine@sha256:abc123)
- /myapp (mycompany/myapp:v1.2.3 → mycompany/myapp:v1.2.4)

Stopped/Removed:
- /nginx (old container)
- /myapp (old container)

All containers updated successfully
```

## Conclusion

Watchtower notifications are essential visibility into your automated update process. Configure Slack or Teams webhooks for real-time team awareness when containers get updated, and use email for formal audit trails. Set `WATCHTOWER_NOTIFICATIONS_LEVEL=info` to avoid noise from successful scans with no updates - you want to be notified about changes, not every poll cycle. Store sensitive credentials like SMTP passwords and webhook tokens as Portainer environment variables rather than hardcoding them in the stack definition.
