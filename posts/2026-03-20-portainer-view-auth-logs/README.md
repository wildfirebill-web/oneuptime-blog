# How to View Authentication Logs in Portainer Business Edition

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Security, Authentication, Audit, BE

Description: Learn how to view and analyze authentication logs in Portainer Business Edition to monitor login activity, detect unauthorized access attempts, and maintain security visibility.

## Introduction

Authentication logs in Portainer Business Edition record every login attempt, session creation, and authentication failure. Monitoring these logs helps detect brute force attacks, unauthorized access attempts, and anomalous login behavior. This guide covers accessing, analyzing, and automating alerts from Portainer's authentication logs.

## Prerequisites

- Portainer Business Edition (BE)
- Admin access to Portainer
- The authentication logging feature is enabled (enabled by default in BE)

## Step 1: Access Authentication Logs in the UI

1. Log into Portainer BE as an administrator.
2. Go to **Settings** in the left sidebar.
3. Click **Authentication logs**.

The logs display:
- **Timestamp**: Exact date and time of the event
- **Username**: Which account was used (or attempted)
- **Origin IP**: Source IP address of the request
- **Event type**: Success, failure, or session expiry
- **User Agent**: Browser/client type (helps identify automated attacks)

## Step 2: Authentication Log Types

```text
LOGIN_SUCCESS     - Successful authentication
LOGIN_FAILURE     - Failed login attempt (wrong password)
LOGIN_UNKNOWN     - Login attempt with unknown username
LOGOUT            - User explicitly logged out
SESSION_EXPIRED   - Session timed out
API_KEY_ACCESS    - Access via API key (not password)
```

## Step 3: View Logs via the API

```bash
PORTAINER_URL="https://portainer.example.com"
TOKEN="your-admin-token"

# Get authentication logs

curl -s -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/auth/logs" | jq .

# Get last 100 authentication events
curl -s -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/auth/logs?limit=100" | jq .

# Filter for failures only
curl -s -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/auth/logs" | \
  jq '[.[] | select(.Type == "failure")]'

# Get failures from a specific IP
SUSPICIOUS_IP="203.0.113.42"
curl -s -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/auth/logs" | \
  jq --arg ip "$SUSPICIOUS_IP" '[.[] | select(.IP == $ip)]'
```

## Step 4: Detect Brute Force Attempts

```bash
#!/bin/bash
# detect-brute-force.sh

PORTAINER_URL="https://portainer.example.com"
TOKEN="your-admin-token"
THRESHOLD=5  # Alert after 5 failures in 10 minutes

RECENT_FAILURES=$(curl -s -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/auth/logs" | \
  jq --argjson window 600 '[.[] | select(.Type == "failure") |
    select(.Timestamp > (now - $window))]')

# Count failures per IP
echo $RECENT_FAILURES | jq -r '.[].IP' | sort | uniq -c | sort -rn | \
  while read COUNT IP; do
    if [ "$COUNT" -ge "$THRESHOLD" ]; then
      echo "BRUTE FORCE ALERT: IP $IP had $COUNT failed logins in the last 10 minutes"

      # Auto-block IP (if using UFW)
      # sudo ufw insert 1 deny from $IP to any comment "Auto-blocked: brute force"
    fi
  done
```

## Step 5: Detect Unusual Login Patterns

```bash
#!/bin/bash
# analyze-auth-patterns.sh

PORTAINER_URL="https://portainer.example.com"
TOKEN="your-admin-token"

LOGS=$(curl -s -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/auth/logs?limit=1000")

echo "=== Authentication Analysis ==="
echo ""

# Unique IPs accessing Portainer
echo "Unique source IPs:"
echo $LOGS | jq -r '.[].IP' | sort -u | while read IP; do
  COUNT=$(echo $LOGS | jq --arg ip "$IP" '[.[] | select(.IP == $ip)] | length')
  echo "  $IP: $COUNT requests"
done

echo ""
echo "Recent failures:"
echo $LOGS | jq -r '.[] | select(.Type == "failure") |
  "  \(.Timestamp | strftime("%Y-%m-%d %H:%M:%S")) | User: \(.Username) | IP: \(.IP)"'

echo ""
echo "After-hours logins (outside 09:00-18:00 UTC):"
echo $LOGS | jq -r '.[] | select(.Type == "success") |
  select((.Timestamp | strftime("%H") | tonumber) < 9 or
         (.Timestamp | strftime("%H") | tonumber) > 18) |
  "  \(.Timestamp | strftime("%Y-%m-%d %H:%M:%S")) | User: \(.Username) | IP: \(.IP)"'
```

## Step 6: Export Authentication Logs for Compliance

```bash
#!/bin/bash
# export-auth-logs-monthly.sh - For compliance reporting

PORTAINER_URL="https://portainer.example.com"
TOKEN="your-admin-token"
YEAR=$(date +%Y)
MONTH=$(date +%m)

# Calculate start and end of last month
START=$(date -d "$YEAR-$MONTH-01 00:00:00" +%s 2>/dev/null || \
        date -j -f "%Y-%m-%d %H:%M:%S" "$YEAR-$MONTH-01 00:00:00" +%s)
END=$(date -d "$YEAR-$MONTH-01 00:00:00 +1 month -1 second" +%s 2>/dev/null || \
      date -j -f "%Y-%m-%d %H:%M:%S" "$YEAR-$MONTH-01 00:00:00" +%s)

REPORT_FILE="auth-logs-${YEAR}-${MONTH}.json"

curl -s -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/auth/logs?after=${START}&before=${END}" > "$REPORT_FILE"

TOTAL=$(cat "$REPORT_FILE" | jq 'length')
FAILURES=$(cat "$REPORT_FILE" | jq '[.[] | select(.Type == "failure")] | length')
SUCCESSES=$(cat "$REPORT_FILE" | jq '[.[] | select(.Type == "success")] | length')

echo "Authentication Report: ${YEAR}-${MONTH}"
echo "  Total events:  $TOTAL"
echo "  Successes:     $SUCCESSES"
echo "  Failures:      $FAILURES"
echo "  Report saved:  $REPORT_FILE"
```

## Step 7: Configure Log Retention

In Portainer BE:
1. Go to **Settings** → **Authentication logs**.
2. Configure **Log retention period** (e.g., 90 days).
3. Enable **Purge logs older than** to automatically remove old logs.

For longer retention, export regularly to external storage:

```bash
# Run weekly export to S3
aws s3 cp auth-logs-$(date +%Y-%m).json s3://your-compliance-bucket/portainer/auth-logs/
```

## Conclusion

Authentication logs in Portainer BE provide essential security visibility into who is accessing your container management platform and when. Monitor these logs regularly for brute force patterns, unusual login times, or access from unknown IPs. Export monthly for compliance reporting and configure alerts to notify your security team of suspicious activity in real time.
