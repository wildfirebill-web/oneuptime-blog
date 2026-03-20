# How to Configure Multiple LDAP Servers for Failover in Portainer (2)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, LDAP, High Availability, Failover, Authentication

Description: Configure Portainer with multiple LDAP servers for high availability authentication failover when the primary server is unavailable.

## Introduction

A single LDAP server is a single point of failure for authentication. If it goes down, no one can log in to Portainer. Configuring multiple LDAP servers provides failover - Portainer tries each server in order until one responds. This is especially important for production environments with uptime requirements.

## How LDAP Failover Works in Portainer

Portainer's LDAP failover:
1. Attempts to connect to the first configured server
2. If connection fails or times out, tries the next server
3. Continues until a server responds or all servers fail
4. Authentication fails only if all servers are unreachable

## Prerequisites

- Multiple LDAP servers (primary + one or more replicas)
- Read-only service account with identical credentials on all servers (or different accounts per server)

## Configuration via the Web UI

In Settings → Authentication → LDAP, you can add multiple servers:

1. Configure the first (primary) server
2. Click **Add server** to add additional servers
3. Each server entry has its own host, port, and optional credentials

## Configuration via API

```bash
TOKEN=$(curl -s -X POST \
  https://portainer.example.com/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"adminpassword"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['jwt'])")

# Configure two LDAP servers

curl -X PUT \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  https://portainer.example.com/api/settings \
  -d '{
    "AuthenticationMethod": 2,
    "ldapsettings": {
      "Servers": [
        {
          "Host": "ldap-primary.example.com",
          "Port": 636,
          "UseTLS": true,
          "SkipVerify": false,
          "Anonymous": false,
          "ReaderDN": "cn=portainer-bind,dc=example,dc=com",
          "Password": "bindpassword"
        },
        {
          "Host": "ldap-secondary.example.com",
          "Port": 636,
          "UseTLS": true,
          "SkipVerify": false,
          "Anonymous": false,
          "ReaderDN": "cn=portainer-bind,dc=example,dc=com",
          "Password": "bindpassword"
        },
        {
          "Host": "ldap-tertiary.example.com",
          "Port": 389,
          "UseTLS": false,
          "StartTLS": true,
          "Anonymous": false,
          "ReaderDN": "cn=portainer-bind,dc=example,dc=com",
          "Password": "bindpassword"
        }
      ],
      "SearchSettings": [
        {
          "BaseDN": "ou=users,dc=example,dc=com",
          "Username": "uid",
          "Filter": "(objectClass=inetOrgPerson)"
        }
      ]
    }
  }'
```

## Active Directory Multi-DC Configuration

For AD environments with multiple domain controllers:

```json
{
  "Servers": [
    {
      "Host": "dc01.corp.example.com",
      "Port": 636,
      "UseTLS": true,
      "ReaderDN": "CN=portainer-svc,OU=Service Accounts,DC=corp,DC=example,DC=com",
      "Password": "servicepassword"
    },
    {
      "Host": "dc02.corp.example.com",
      "Port": 636,
      "UseTLS": true,
      "ReaderDN": "CN=portainer-svc,OU=Service Accounts,DC=corp,DC=example,DC=com",
      "Password": "servicepassword"
    }
  ]
}
```

**Pro tip**: Active Directory environments can also use the domain's DNS name which resolves to multiple DCs:
```text
Host: corp.example.com
Port: 636
```

This lets Windows DNS SRV records handle failover transparently.

## Monitoring LDAP Server Health

Set up monitoring to detect LDAP failures before they impact users:

```bash
#!/bin/bash
# check-ldap-health.sh

LDAP_SERVERS=("ldap-primary.example.com:636" "ldap-secondary.example.com:636")
BIND_DN="cn=portainer-bind,dc=example,dc=com"
BIND_PW="bindpassword"

for server in "${LDAP_SERVERS[@]}"; do
  HOST=$(echo $server | cut -d: -f1)
  PORT=$(echo $server | cut -d: -f2)

  # Try to bind
  RESULT=$(ldapsearch -x \
    -H "ldaps://${HOST}:${PORT}" \
    -D "$BIND_DN" \
    -w "$BIND_PW" \
    -b "dc=example,dc=com" \
    -s base "(objectClass=*)" \
    2>&1)

  if echo "$RESULT" | grep -q "result: 0 Success"; then
    echo "✓ ${HOST}:${PORT} - HEALTHY"
  else
    echo "✗ ${HOST}:${PORT} - UNREACHABLE or AUTH FAILURE"
    echo "  Error: $(echo "$RESULT" | grep "ldap_" | head -1)"
    # Send alert here
  fi
done
```

## Connection Timeout Configuration

Portainer uses a default connection timeout when trying each LDAP server. If servers are slow to respond, users may experience delays. Ensure your LDAP servers are in the same datacenter or have low latency.

## Testing Failover

```bash
# Simulate primary server failure
# Block ldap-primary traffic temporarily
sudo iptables -A OUTPUT -d ldap-primary.example.com -j DROP

# Try to log in to Portainer - should succeed via secondary

# Restore connectivity
sudo iptables -D OUTPUT -d ldap-primary.example.com -j DROP
```

## Conclusion

Multiple LDAP server configuration in Portainer is straightforward - add each server to the `Servers` array in order of preference. Portainer handles failover automatically, trying each server until one responds. For production environments, at least two LDAP servers should be configured, ideally in separate availability zones or data centers.
