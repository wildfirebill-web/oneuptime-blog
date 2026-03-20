# How to Audit User Activity in Portainer Business Edition

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Business Edition, Audit Logs, Security, Compliance

Description: Learn how to access and interpret user activity audit logs in Portainer Business Edition for compliance and security monitoring.

## What Are Portainer Audit Logs?

Portainer Business Edition includes comprehensive audit logging that records:

- User login and logout events.
- Container start, stop, delete operations.
- Stack deployments and updates.
- Registry additions and modifications.
- User and team management changes.
- Environment configuration changes.

## Accessing Audit Logs in Portainer BE

1. Log in to Portainer as an administrator.
2. Go to **Settings > Authentication logs** or **Logs** (depends on version).
3. View the chronological activity feed.

## What Each Log Entry Contains

```json
{
  "timestamp": "2026-03-20T14:32:45Z",
  "username": "alice.smith",
  "userRole": "Standard User",
  "context": "endpoint/production",
  "action": "Container Start",
  "resourceType": "Container",
  "resourceId": "abc123def456",
  "payload": "{\"containerName\": \"my-app\"}",
  "result": "success"
}
```

## Filtering Audit Logs

Use the filter options in Portainer to narrow down logs by:

- **Username**: Track specific user actions.
- **Time range**: Focus on a specific period.
- **Action type**: Filter by operation (start, stop, deploy).
- **Environment**: Focus on a specific cluster.

## Exporting Audit Logs via API

```bash
# Export audit logs via the Portainer API

curl -s "https://portainer.mycompany.com/api/auth/logs" \
  -H "Authorization: Bearer ${ADMIN_TOKEN}" | jq '.'

# Filter by username
curl -s "https://portainer.mycompany.com/api/auth/logs?username=alice" \
  -H "Authorization: Bearer ${ADMIN_TOKEN}" | jq '.'

# Export all logs to a file
curl -s "https://portainer.mycompany.com/api/logs" \
  -H "Authorization: Bearer ${ADMIN_TOKEN}" \
  -o portainer-audit-$(date +%Y%m%d).json
```

## Setting Up External Audit Log Forwarding

For long-term retention, forward Portainer logs to an external system:

```bash
# Forward Portainer container logs to syslog
docker run -d \
  --name portainer \
  --log-driver syslog \
  --log-opt syslog-address=udp://siem.mycompany.com:514 \
  --log-opt tag="portainer" \
  portainer/portainer-be:latest
```

## Automating Compliance Reports

```bash
#!/bin/bash
# Generate weekly security report

PORTAINER_URL="https://portainer.mycompany.com"
API_TOKEN="${PORTAINER_API_TOKEN}"
WEEK_AGO=$(date -d "7 days ago" +%Y-%m-%dT%H:%M:%SZ 2>/dev/null || \
           date -v-7d +%Y-%m-%dT%H:%M:%SZ)

# Get all admin actions in the past week
ADMIN_ACTIONS=$(curl -s "${PORTAINER_URL}/api/logs?since=${WEEK_AGO}&role=admin" \
  -H "Authorization: Bearer ${API_TOKEN}")

# Count by action type
echo "=== Admin Actions This Week ==="
echo "$ADMIN_ACTIONS" | jq '[.[] | .action] | group_by(.) | .[] | {action: .[0], count: length}'

# Count failed login attempts
echo "=== Failed Login Attempts ==="
curl -s "${PORTAINER_URL}/api/auth/logs?since=${WEEK_AGO}" \
  -H "Authorization: Bearer ${API_TOKEN}" | \
  jq '[.[] | select(.result == "failure")] | length'
```

## Key Audit Events to Monitor

| Event | Security Concern |
|-------|-----------------|
| Multiple failed logins | Brute force attempt |
| New admin user created | Privilege escalation |
| Environment deleted | Accidental/malicious deletion |
| Registry credentials changed | Credential compromise |
| Security settings changed | Policy bypass attempt |

## Conclusion

Portainer Business Edition's audit logs provide the visibility needed for compliance (SOC 2, ISO 27001, PCI DSS) and security incident investigation. Set up regular log review processes and forward logs to your SIEM for long-term retention and automated alerting.
