# How to Configure Activity Logs in Portainer Business Edition

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Business Edition, Activity Logs, Auditing, Compliance, Docker

Description: Learn how to configure and use activity logs in Portainer Business Edition to track user actions, container changes, and stack deployments for security auditing.

---

Portainer Business Edition includes an activity logging system that records every user action: who deployed a stack, who restarted a container, and who changed settings. This is essential for compliance and incident investigation.

## Prerequisites

- Portainer Business Edition with a valid license
- Admin access to Portainer

## Enabling Activity Logging

Activity logging is enabled by default in Business Edition. Verify it is active:

1. Go to **Settings > General**.
2. Under **Activity Logging**, confirm the feature is enabled.
3. Set the **Retention Period** (e.g., 90 days).

## What Gets Logged

| Action | Logged Data |
|---|---|
| User login | Username, IP, timestamp, success/failure |
| Container start/stop/restart | User, container name, environment |
| Stack deploy/update | User, stack name, compose diff |
| Settings changes | User, setting changed, old/new value |
| User management | Admin, action, affected user |
| Environment changes | User, environment, change type |

## Viewing Activity Logs

In Portainer go to **Settings > Authentication Logs** or the dedicated **Logs** section:

1. Select a date range.
2. Filter by user, environment, or action type.
3. Export logs as CSV for external analysis.

## Streaming to an External System

For SIEM integration, stream activity logs to syslog:

```bash
# Start Portainer with syslog logging enabled
docker run -d \
  --name portainer \
  --log-driver syslog \
  --log-opt syslog-address=udp://syslog-server:514 \
  --log-opt syslog-facility=daemon \
  --log-opt tag="portainer" \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ee:latest
```

## Querying Logs via API

```bash
# Get activity logs via the Portainer API
TOKEN=$(curl -s -X POST http://localhost:9000/api/auth \
  -H "Content-Type: application/json" \
  -d '{"Username":"admin","Password":"yourpassword"}' | jq -r .jwt)

# Query activity logs
curl -H "Authorization: Bearer $TOKEN" \
  "http://localhost:9000/api/logs/activity?limit=100&after=1709000000"
```

## Log Retention Configuration

Set up automatic log cleanup:

1. In Portainer go to **Settings > General**.
2. Under **Activity Logging**, set **Log retention period** to the desired number of days.
3. Logs older than the retention period are automatically deleted.

For compliance requirements, export and archive logs before the retention period expires using the CSV export feature or API.
