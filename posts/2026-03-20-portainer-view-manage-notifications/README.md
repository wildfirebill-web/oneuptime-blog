# How to View and Manage Notifications in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Notifications, UI, Administration, Events, Docker

Description: Learn how to view, manage, and clear notifications in Portainer, including understanding notification types and configuring alert preferences.

---

Portainer's notification system captures important events - container failures, stack deployment results, agent connectivity changes - and displays them in the notification bell in the top navigation bar.

## Accessing Notifications

Click the bell icon in the Portainer top navigation bar to open the notification panel. The number badge shows unread notifications.

## Notification Types

| Type | Trigger |
|---|---|
| Info | Stack deployed successfully, container started |
| Warning | Agent connection degraded, snapshot delayed |
| Error | Stack deployment failed, container crashed, registry pull failed |

## Viewing Notification Details

Each notification shows:

- **Timestamp**: When the event occurred
- **Message**: Description of the event
- **Environment**: Which Docker/Kubernetes environment was affected
- **Action**: Link to the relevant resource (stack, container, etc.)

## Marking Notifications as Read

Click a notification to mark it as read. The badge count decreases. Click the **Mark all as read** option at the top of the notification panel to clear all unread indicators at once.

## Clearing Notifications

```bash
# Via Portainer UI:

# 1. Open the notification panel
# 2. Click "Clear all" to remove all notifications

# Via API:
TOKEN="ptr_xxxx"
curl -X DELETE \
  -H "X-API-Key: $TOKEN" \
  http://localhost:9000/api/notifications
```

## Notification Retention and Performance

A large notification backlog can slow down Portainer's UI. In high-activity environments, clear notifications regularly:

1. In Portainer go to the notification bell.
2. Click **Clear all**.

For automated cleanup, compact the database periodically:

```bash
docker stop portainer
docker run --rm -v portainer_data:/data portainer/portainer-ce:latest --compact-db
docker start portainer
```

## Webhook Notifications (Business Edition)

In Portainer Business Edition, configure outbound webhooks to send notifications to Slack, Teams, or a custom endpoint:

1. Go to **Settings > General > Notifications**.
2. Add a webhook URL.
3. Select which event types trigger notifications.

```bash
# Example Slack webhook payload sent by Portainer BE:
{
  "event": "stack.deployment.failed",
  "stack": "myapp",
  "environment": "production",
  "timestamp": "2026-03-20T10:00:00Z",
  "details": "Image pull failed: myimage:1.2.3 not found"
}
```
