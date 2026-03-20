# How to View Authentication Logs in Portainer Business Edition

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Business Edition, Authentication, Security Logs, Compliance

Description: Learn how to access and analyze authentication logs in Portainer Business Edition to monitor login activity and detect unauthorized access attempts.

## Authentication Logs Overview

Portainer Business Edition records all authentication events:

- Successful logins (with timestamp, IP, and username).
- Failed login attempts (wrong password, unknown user).
- Logout events.
- Token-based authentication (API access tokens).
- Session expirations.

## Accessing Authentication Logs

1. Log in to Portainer BE as an administrator.
2. Go to **Settings** in the left sidebar.
3. Click **Authentication logs**.
4. View the list of authentication events.

## What's Shown in the Log

Each entry shows:
- **Timestamp**: When the event occurred.
- **Username**: Who attempted to authenticate.
- **Authentication context**: Login method (form, OAuth, LDAP, API token).
- **Status**: Success or failure.
- **Reason**: For failures (wrong password, account not found).
- **Source IP**: From where the request originated.

## Filtering Authentication Logs

In the Portainer UI:
- Filter by **username** to track a specific user's login history.
- Filter by **status** to see only failures (for security review).
- Filter by **date range** for incident investigation.

## Retrieving Authentication Logs via API

```bash
# Get all authentication logs
curl -s "https://portainer.mycompany.com/api/auth/logs" \
  -H "Authorization: Bearer ${ADMIN_TOKEN}" | \
  jq '[.[] | {
    time: .Timestamp,
    user: .Username,
    status: .Result,
    ip: .SourceIPAddress
  }]'

# Get only failed attempts
curl -s "https://portainer.mycompany.com/api/auth/logs" \
  -H "Authorization: Bearer ${ADMIN_TOKEN}" | \
  jq '[.[] | select(.Result == "failure")]'
```

## Detecting Brute Force Attacks

```bash
#!/bin/bash
# Alert on accounts with multiple failed login attempts

THRESHOLD=5  # Alert if more than 5 failures in the last hour

curl -s "https://portainer.mycompany.com/api/auth/logs" \
  -H "Authorization: Bearer ${ADMIN_TOKEN}" | \
  jq --argjson threshold "$THRESHOLD" '
    [.[] | select(.Result == "failure")] |
    group_by(.Username) |
    .[] |
    select(length >= $threshold) |
    {
      username: .[0].Username,
      failures: length,
      first_attempt: .[0].Timestamp,
      last_attempt: .[-1].Timestamp,
      ips: [.[].SourceIPAddress] | unique
    }
  ' | jq '.'
```

## Monitoring Authentication Events in Real-Time

```bash
# Stream Portainer logs and filter for auth events
docker logs -f portainer 2>&1 | \
  grep -E "login|logout|authenticated|failed" | \
  while read line; do
    echo "[$(date +%H:%M:%S)] $line"
    # Optionally send to Slack or PagerDuty
  done
```

## Authentication Log Retention

Configure how long authentication logs are retained:

1. Go to **Settings > Security**.
2. Find **Authentication log retention**.
3. Set the retention period (e.g., 90 days for compliance).

## Integrating with SIEM

Forward authentication events to your Security Information and Event Management (SIEM) system:

```bash
# Forward Portainer auth logs to Splunk via HTTP Event Collector
curl -s "https://portainer.mycompany.com/api/auth/logs?since=${LAST_EXPORT}" \
  -H "Authorization: Bearer ${ADMIN_TOKEN}" | \
  jq -c '.[]' | while read event; do
    curl -s -X POST "https://splunk.mycompany.com:8088/services/collector" \
      -H "Authorization: Splunk ${SPLUNK_HEC_TOKEN}" \
      -d "{\"sourcetype\": \"portainer:auth\", \"event\": ${event}}"
  done
```

## Conclusion

Authentication logs in Portainer Business Edition are a critical security asset. Review them regularly for suspicious patterns, integrate with your SIEM for automated alerting, and maintain sufficient retention for compliance requirements.
