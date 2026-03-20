# How to Configure Multiple LDAP Servers for Failover in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, LDAP, High Availability, Authentication, Business Edition

Description: Learn how to configure multiple LDAP servers in Portainer Business Edition for redundancy and failover authentication.

---

In production environments, a single LDAP server is a single point of failure for authentication. Portainer Business Edition supports configuring multiple LDAP servers so login continues if the primary server is unavailable.

## Prerequisites

- Portainer Business Edition
- At least two LDAP servers (primary and secondary/replica)
- Admin credentials for Portainer

## Configure Multiple LDAP Servers via UI

1. Log in as administrator
2. Navigate to **Settings > Authentication**
3. Select **LDAP** as the authentication method
4. Fill in the primary LDAP server details
5. Look for the **Additional LDAP Servers** or **Fallback** section to add secondary servers

## Configure via API

```bash
# Authenticate

TOKEN=$(curl -s -X POST \
  https://localhost:9443/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"yourpassword"}' \
  --insecure | python3 -c "import sys,json; print(json.load(sys.stdin)['jwt'])")

# Configure LDAP with multiple servers
curl -X PUT \
  https://localhost:9443/api/settings \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "AuthenticationMethod": 2,
    "LDAPSettings": {
      "AnonymousMode": false,
      "ReaderDN": "cn=portainer,dc=example,dc=com",
      "Password": "ldap-reader-password",
      "URLs": [
        "ldaps://ldap1.example.com:636",
        "ldaps://ldap2.example.com:636"
      ],
      "TLSConfig": {
        "TLS": true,
        "TLSSkipVerify": false
      },
      "SearchSettings": [
        {
          "BaseDN": "dc=example,dc=com",
          "Filter": "(objectClass=person)",
          "UserNameAttribute": "sAMAccountName"
        }
      ],
      "AutoCreateUsers": true
    }
  }' \
  --insecure
```

## Load Balancer Approach

A more robust approach is to put LDAP servers behind a load balancer:

```bash
# Configure Portainer to use a load balancer VIP
# The load balancer handles failover transparently
curl -X PUT \
  https://localhost:9443/api/settings \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "AuthenticationMethod": 2,
    "LDAPSettings": {
      "URLs": ["ldaps://ldap-vip.example.com:636"],
      "ReaderDN": "cn=portainer,dc=example,dc=com",
      "Password": "ldap-reader-password",
      "TLSConfig": {"TLS": true, "TLSSkipVerify": false},
      "SearchSettings": [{
        "BaseDN": "dc=example,dc=com",
        "Filter": "(objectClass=person)",
        "UserNameAttribute": "sAMAccountName"
      }]
    }
  }' \
  --insecure
```

## Test LDAP Connectivity

After configuration, test with each server URL individually:

```bash
# Test primary LDAP server reachability
ldapsearch -H ldaps://ldap1.example.com:636 \
  -D "cn=portainer,dc=example,dc=com" \
  -w "ldap-reader-password" \
  -b "dc=example,dc=com" \
  "(sAMAccountName=testuser)" \
  sAMAccountName

# Test secondary server
ldapsearch -H ldaps://ldap2.example.com:636 \
  -D "cn=portainer,dc=example,dc=com" \
  -w "ldap-reader-password" \
  -b "dc=example,dc=com" \
  "(sAMAccountName=testuser)" \
  sAMAccountName
```

## Failover Behavior

Portainer tries LDAP servers in the order listed. If the first server fails (connection timeout), it automatically tries the next server. The timeout before failover is typically 5-10 seconds.

---

*Monitor LDAP server availability alongside Portainer with [OneUptime](https://oneuptime.com).*
