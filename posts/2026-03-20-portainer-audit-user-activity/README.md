# How to Audit User Activity in Portainer Business Edition

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Security, Audit, Compliance, BE

Description: Learn how to access and analyze user activity audit logs in Portainer Business Edition to track who did what, when, and from where in your container infrastructure.

## Introduction

Portainer Business Edition includes comprehensive activity logging that records all user actions — container deployments, configuration changes, login events, and more. This audit trail is essential for security investigations, compliance requirements (SOC2, PCI-DSS, HIPAA), and understanding what changed before an incident.

## Prerequisites

- Portainer Business Edition (BE)
- Admin access to Portainer
- Understanding of what events you want to audit

## What Gets Logged

Portainer BE logs the following event types:

| Category | Events |
|----------|--------|
| Authentication | Login, logout, failed logins |
| Users | Create, update, delete users |
| Teams | Create, update, delete teams; membership changes |
| Environments | Add, update, remove environments |
| Stacks | Deploy, update, delete stacks |
| Containers | Create, start, stop, remove containers |
| Images | Pull, remove images |
| Volumes | Create, remove volumes |
| Networks | Create, remove networks |
| Settings | Any configuration changes |
| Registries | Add, update, delete registries |

## Step 1: Access Activity Logs in the UI

1. Log into Portainer BE as an administrator.
2. Click **Settings** in the left sidebar.
3. Click **Activity logs** or **Audit logs**.
4. The log view shows:
   - **Timestamp**: When the action occurred
   - **User**: Who performed the action
   - **Action**: What was done
   - **Context**: Which environment/resource
   - **IP Address**: Source IP of the request

## Step 2: Filter Audit Logs

Apply filters to find specific events:

1. **Time range**: Filter to a specific date/time window
2. **User**: Show actions by a specific user
3. **Action type**: Filter by authentication, deployment, etc.
4. **Environment**: Show activity in specific environments

```bash
# Access logs via Portainer API
TOKEN="your-admin-token"
PORTAINER_URL="https://portainer.example.com"

# Get activity logs
curl -s -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/useractivity?limit=100&start=1" | jq .

# Filter by user
curl -s -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/useractivity?limit=100&userID=5" | jq .

# Filter by time range (Unix timestamps)
START_TIME=1711670400  # 2024-03-29 00:00:00 UTC
END_TIME=1711756800    # 2024-03-30 00:00:00 UTC

curl -s -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/useractivity?limit=100&after=${START_TIME}&before=${END_TIME}" | jq .
```

## Step 3: Export Audit Logs

### Via the UI

1. In the Activity logs view, click **Export**.
2. Select CSV or JSON format.
3. Choose the date range.
4. Download the file.

### Via the API

```bash
#!/bin/bash
# export-audit-logs.sh

PORTAINER_URL="https://portainer.example.com"
TOKEN="your-admin-token"
OUTPUT_FILE="audit-logs-$(date +%Y%m%d).json"

# Calculate last 30 days
END_TIME=$(date +%s)
START_TIME=$((END_TIME - 30 * 24 * 3600))

echo "Exporting audit logs from last 30 days..."

# Paginate through all logs
PAGE=1
LIMIT=500
ALL_LOGS=()

while true; do
  OFFSET=$(( (PAGE - 1) * LIMIT + 1 ))

  RESPONSE=$(curl -s -H "Authorization: Bearer $TOKEN" \
    "${PORTAINER_URL}/api/useractivity?limit=${LIMIT}&start=${OFFSET}&after=${START_TIME}&before=${END_TIME}")

  COUNT=$(echo $RESPONSE | jq 'length')

  if [ "$COUNT" -eq 0 ]; then
    break
  fi

  echo "Retrieved page $PAGE ($COUNT records)..."
  echo $RESPONSE >> "$OUTPUT_FILE.tmp"
  PAGE=$((PAGE + 1))

  if [ "$COUNT" -lt "$LIMIT" ]; then
    break
  fi
done

cat "$OUTPUT_FILE.tmp" | jq -s 'add' > "$OUTPUT_FILE"
rm -f "$OUTPUT_FILE.tmp"

TOTAL=$(cat "$OUTPUT_FILE" | jq 'length')
echo "Exported $TOTAL audit log entries to $OUTPUT_FILE"
```

## Step 4: Audit Log Analysis

```bash
# Find all failed login attempts
curl -s -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/useractivity" | \
  jq '[.[] | select(.Type == "auth.login.failure")] | length'

# List unique users who deleted containers
curl -s -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/useractivity" | \
  jq '[.[] | select(.Action | contains("DELETE")) | .Username] | unique'

# Find settings changes in last 7 days
WEEK_AGO=$(( $(date +%s) - 7 * 24 * 3600 ))
curl -s -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/useractivity?after=${WEEK_AGO}" | \
  jq '[.[] | select(.Action | contains("settings"))]'
```

## Step 5: Alert on Suspicious Activity

```bash
#!/bin/bash
# detect-suspicious-activity.sh — Run periodically

PORTAINER_URL="https://portainer.example.com"
TOKEN="your-admin-token"
SLACK_WEBHOOK="https://hooks.slack.com/services/YOUR/WEBHOOK"

# Get logs from the last hour
ONE_HOUR_AGO=$(( $(date +%s) - 3600 ))

RECENT_LOGS=$(curl -s -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/useractivity?after=${ONE_HOUR_AGO}")

# Check for multiple failed logins (brute force indicator)
FAILED_LOGINS=$(echo $RECENT_LOGS | jq '[.[] | select(.Type == "auth.login.failure")] | length')

if [ "$FAILED_LOGINS" -gt 10 ]; then
  MESSAGE="ALERT: $FAILED_LOGINS failed login attempts in the last hour on Portainer"
  curl -s -X POST "$SLACK_WEBHOOK" \
    -H "Content-Type: application/json" \
    -d "{\"text\": \"$MESSAGE\"}"
fi

# Check for admin deletions
ADMIN_DELETIONS=$(echo $RECENT_LOGS | \
  jq '[.[] | select(.Action | contains("DELETE")) | select(.Role == "administrator")] | length')

if [ "$ADMIN_DELETIONS" -gt 5 ]; then
  MESSAGE="ALERT: $ADMIN_DELETIONS admin deletion actions in the last hour"
  curl -s -X POST "$SLACK_WEBHOOK" \
    -H "Content-Type: application/json" \
    -d "{\"text\": \"$MESSAGE\"}"
fi
```

## Step 6: Ship Logs to a SIEM

Forward Portainer audit logs to your security information and event management system:

```bash
# Option 1: Docker log driver to syslog
docker run -d \
  --name portainer \
  --log-driver syslog \
  --log-opt syslog-address=tcp://siem.company.com:514 \
  --log-opt tag=portainer \
  -v /var/run/docker.sock:/var/run/docker.sock \
  portainer/portainer-be:latest

# Option 2: Stream API logs to SIEM via Filebeat/Fluentd
# Configure Filebeat to call Portainer API periodically and ship to Elasticsearch
```

## Conclusion

Portainer BE's activity logs provide complete visibility into all actions taken within your container management platform. Use the UI for interactive investigation, the API for programmatic access and automated reporting, and set up alerts for suspicious activity patterns. Export logs regularly to your SIEM system to maintain a complete audit trail for compliance requirements.
