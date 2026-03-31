# How to View Authentication Logs in Portainer Business

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Business Edition, Authentication, Audit Log, Security, Compliance

Description: Learn how to view and analyze authentication logs in Portainer Business Edition to track login attempts, identify unauthorized access, and meet audit compliance requirements.

---

Portainer Business Edition logs every authentication event - successful logins, failed attempts, token usage, and OAuth flows. These logs are essential for security auditing and identifying brute-force attacks or compromised accounts.

## Accessing Authentication Logs

1. Log in to Portainer with an admin account.
2. Go to **Settings > Logs** (or **Authentication Logs** depending on version).
3. Filter by date range, user, or event type.

## What Is Logged

| Event | Logged Data |
|---|---|
| Successful login | Username, source IP, method (local/LDAP/OAuth), timestamp |
| Failed login | Username, source IP, failure reason, timestamp |
| API token usage | Token name, source IP, endpoint accessed |
| OAuth flow | User, provider, success/failure |
| Session expiry | Username, session duration |
| Password change | Username, timestamp |

## Filtering Authentication Logs

```bash
# Via API - get authentication logs

TOKEN="ptr_xxxx"

curl -H "X-API-Key: $TOKEN" \
  "http://localhost:9000/api/logs/auth?limit=100&offset=0" | \
  jq '.logs[] | {user: .username, ip: .ip, event: .action, time: .timestamp}'
```

## Detecting Brute Force Attacks

Look for rapid repeated failures from the same IP:

```bash
# Find IPs with multiple failed logins in the last hour
curl -H "X-API-Key: $TOKEN" \
  "http://localhost:9000/api/logs/auth?action=login.failed&limit=1000" | \
  jq -r '.logs[].ip' | sort | uniq -c | sort -rn | head -20

# IPs with counts >> 10 are suspicious
```

## Setting Up Log Retention

1. Go to **Settings > General**.
2. Under **Activity Logging**, set **Log retention period**.
3. Recommended: 90 days for most compliance frameworks (PCI-DSS requires 1 year).

## Exporting for SIEM

Export authentication logs regularly for external analysis:

```bash
# Export last 30 days of auth logs to CSV
FROM=$(date -d "30 days ago" +%s)
TO=$(date +%s)

curl -H "X-API-Key: $TOKEN" \
  "http://localhost:9000/api/logs/auth?after=$FROM&before=$TO&limit=10000" | \
  jq -r '.logs[] | [.timestamp, .username, .ip, .action, .context] | @csv' > auth-logs.csv
```

## Real-time Monitoring with OneUptime

Configure OneUptime to periodically check for authentication failures via the Portainer API. Create a monitor that calls `/api/logs/auth?action=login.failed` and alerts if the count exceeds a threshold in a rolling time window.
