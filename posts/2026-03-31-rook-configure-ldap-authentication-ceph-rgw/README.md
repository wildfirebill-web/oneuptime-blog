# How to Configure LDAP Authentication for Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RGW, LDAP, Authentication, Security

Description: Learn how to configure LDAP authentication for Ceph RGW, enabling users to authenticate to S3 object storage using their existing directory service credentials.

---

## Overview

Ceph RGW supports LDAP-based authentication, allowing users to authenticate using their existing corporate directory credentials (Active Directory, OpenLDAP) rather than separate RGW access keys. RGW maps LDAP users to internal RGW users, making it easier to manage access at scale using existing identity infrastructure.

## How LDAP Authentication Works in RGW

RGW's LDAP authentication works as follows:
1. Client sends an HTTP request with an Authorization header containing an LDAP username and password
2. RGW binds to the LDAP server using the provided credentials
3. If the LDAP bind succeeds, RGW creates or maps a local user
4. The request proceeds with the mapped user's permissions

This is primarily used with the Swift API, where HTTP Basic Auth is supported.

## Installing Required Packages

```bash
# On the RGW host, install LDAP libraries
apt-get install libldap2-dev  # Debian/Ubuntu
dnf install openldap-devel    # RHEL/CentOS
```

## Configuring RGW for LDAP

```bash
# Set LDAP server URI
ceph config set client.rgw rgw_ldap_uri "ldap://ldap-host:389"

# Set bind DN (service account for searching)
ceph config set client.rgw rgw_ldap_binddn "cn=ceph-bind,ou=services,dc=example,dc=com"

# Set bind password (store in a file for security)
echo -n "service-account-password" > /etc/ceph/ldap_bind_password
chmod 600 /etc/ceph/ldap_bind_password
ceph config set client.rgw rgw_ldap_bindpw_path /etc/ceph/ldap_bind_password

# Set search base DN
ceph config set client.rgw rgw_ldap_searchdn "ou=users,dc=example,dc=com"

# Set search filter
ceph config set client.rgw rgw_ldap_dnattr "uid"

# Optional: require users to be in a specific LDAP group
ceph config set client.rgw rgw_ldap_searchfilter "(memberOf=cn=ceph-users,ou=groups,dc=example,dc=com)"
```

## Using LDAP Authentication via Swift API

Clients use the LDAP username and password with Swift auth:

```bash
# Authenticate using LDAP credentials
curl -i http://rgw-host:80/auth/1.0 \
  -H "X-Auth-User: alice" \
  -H "X-Auth-Key: alice-ldap-password"

# Use the returned token for subsequent requests
TOKEN=$(curl -s http://rgw-host:80/auth/1.0 \
  -H "X-Auth-User: alice" \
  -H "X-Auth-Key: alice-ldap-password" \
  -D - | grep X-Auth-Token | awk '{print $2}')

# List containers
curl http://rgw-host:80/swift/v1/ \
  -H "X-Auth-Token: $TOKEN"
```

## Using swiftclient with LDAP

```bash
swift --auth http://rgw-host:80/auth/1.0 \
  --user alice \
  --key alice-ldap-password \
  list
```

## LDAP over TLS

For production, use LDAPS or STARTTLS:

```bash
# Use LDAPS
ceph config set client.rgw rgw_ldap_uri "ldaps://ldap-host:636"

# Configure TLS CA certificate
ceph config set client.rgw rgw_ldap_cacert /etc/ceph/ldap-ca.crt

# Or use STARTTLS
ceph config set client.rgw rgw_ldap_uri "ldap://ldap-host:389"
ceph config set client.rgw rgw_ldap_use_start_tls true
```

## Active Directory Configuration

For Active Directory, adjust the search attributes:

```bash
# AD uses sAMAccountName for user login names
ceph config set client.rgw rgw_ldap_dnattr "sAMAccountName"

# AD search base
ceph config set client.rgw rgw_ldap_searchdn "DC=corp,DC=example,DC=com"

# AD bind DN format
ceph config set client.rgw rgw_ldap_binddn "corp\\service-account"
```

## Verifying LDAP Configuration

Test LDAP connectivity from the RGW host:

```bash
# Test LDAP bind
ldapsearch -H ldap://ldap-host:389 \
  -D "cn=ceph-bind,ou=services,dc=example,dc=com" \
  -w service-account-password \
  -b "ou=users,dc=example,dc=com" \
  "(uid=alice)"

# Check RGW logs for LDAP auth attempts
grep "ldap" /var/log/ceph/ceph-client.rgw.*.log
```

## Summary

Ceph RGW LDAP authentication integrates with existing directory services so users can access S3/Swift with their corporate credentials. Configure the LDAP server URI, bind credentials, and search base in Ceph config, then users authenticate via the Swift auth endpoint. Use LDAPS or STARTTLS in production, and filter access by LDAP group membership for fine-grained control over who can access RGW.
