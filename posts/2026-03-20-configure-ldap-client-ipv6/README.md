# How to Configure LDAP Client Connections over IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: LDAP, IPv6, ldap.conf, SSSD, nslcd, Authentication, Linux

Description: Configure Linux LDAP clients to connect to directory servers over IPv6 using ldap.conf, SSSD, and nslcd for centralized authentication on IPv6 networks.

---

LDAP clients on Linux — whether using ldap-utils, SSSD, or nslcd — need specific configuration to connect to LDAP servers over IPv6. The key difference from IPv4 is the bracket notation for IPv6 addresses in LDAP URIs.

## LDAP URI Format for IPv6

IPv6 addresses in LDAP URIs require bracket notation:

```
ldap://[2001:db8::1]:389/dc=example,dc=com
ldaps://[2001:db8::1]:636/dc=example,dc=com
```

## Configuring ldap.conf for IPv6

The global LDAP client configuration at `/etc/ldap/ldap.conf` (or `/etc/openldap/ldap.conf`):

```bash
# /etc/ldap/ldap.conf

# LDAP server URI with IPv6 address (brackets required)
URI ldap://[2001:db8::1] ldap://[2001:db8::2]

# Or using a hostname with AAAA record
URI ldap://ldap.example.com

# Base DN for searches
BASE dc=example,dc=com

# TLS settings
TLS_CACERT /etc/ssl/certs/ca-certificates.crt
TLS_REQCERT demand

# Optional: bind credentials (usually not here for security)
# BINDDN cn=ldapbind,dc=example,dc=com
# BINDPW secret
```

## Testing LDAP Connection over IPv6

```bash
# Test basic connectivity with ldapsearch
ldapsearch -H ldap://[2001:db8::1] \
  -x \
  -b "dc=example,dc=com" \
  "(objectClass=posixAccount)" uid cn

# Test with authentication
ldapsearch -H ldap://[2001:db8::1] \
  -D "cn=admin,dc=example,dc=com" \
  -w "adminpassword" \
  -b "dc=example,dc=com" \
  "(uid=testuser)"

# Test LDAPS (TLS) over IPv6
ldapsearch -H ldaps://[2001:db8::1]:636 \
  -x \
  -b "dc=example,dc=com" \
  "(objectClass=*)" | head -20
```

## Configuring SSSD for LDAP over IPv6

SSSD is the modern way to configure LDAP authentication on Linux:

```ini
# /etc/sssd/sssd.conf

[sssd]
services = nss, pam, sudo
config_file_version = 2
domains = LDAP

[domain/LDAP]
id_provider = ldap
auth_provider = ldap

# LDAP server URI with IPv6 - brackets required
ldap_uri = ldap://[2001:db8::1]

# Fallback to second server
ldap_backup_uri = ldap://[2001:db8::2]

# Base DN
ldap_search_base = dc=example,dc=com

# Bind credentials
ldap_default_bind_dn = cn=sssd-bind,dc=example,dc=com
ldap_default_authtok = sssd_bind_password

# TLS
ldap_tls_cacertdir = /etc/ssl/certs
ldap_tls_reqcert = demand

# User and group object classes
ldap_schema = rfc2307
ldap_user_object_class = posixAccount
ldap_group_object_class = posixGroup

# Cache settings
cache_credentials = true
enumerate = false
```

```bash
# Set permissions and restart SSSD
sudo chmod 600 /etc/sssd/sssd.conf
sudo systemctl restart sssd

# Test user lookup
id testuser
getent passwd testuser

# Test authentication
su - testuser
```

## Configuring nslcd for LDAP over IPv6

```bash
# /etc/nslcd.conf

# LDAP server URI with IPv6
uri ldap://[2001:db8::1]/

# Base DN
base dc=example,dc=com

# Bind credentials
binddn cn=nslcd-bind,dc=example,dc=com
bindpw bind_password

# TLS
ssl start_tls
tls_cacertfile /etc/ssl/certs/ca-certificates.crt
tls_reqcert demand
```

## Configuring PAM for LDAP Authentication

```bash
# /etc/pam.d/common-auth
auth    required        pam_env.so
auth    sufficient      pam_unix.so nullok_secure
auth    requisite       pam_succeed_if.so uid >= 1000 quiet_success
auth    sufficient      pam_sss.so use_first_pass  # or pam_ldap.so
auth    required        pam_deny.so

# /etc/nsswitch.conf
passwd: files sss   # or files ldap
group:  files sss
shadow: files sss
```

## Debugging LDAP IPv6 Client Issues

```bash
# Test with verbose ldapsearch
LDAPTLS_REQCERT=never ldapsearch -H ldap://[2001:db8::1] \
  -x -d 7 -b "dc=example,dc=com" "(cn=admin)" 2>&1 | head -50

# Check SSSD logs
sudo journalctl -u sssd -f
sudo sssd --logger=stderr -d 7 -i  # Debug mode

# Test NSS resolution
getent passwd  # Should include LDAP users if configured

# Check if LDAP port is reachable
nc -6 -w 3 2001:db8::1 389 && echo "Connected"
```

Configuring LDAP clients for IPv6 requires using bracket notation in LDAP URIs and verifying that your SSSD or nslcd configuration correctly targets IPv6-accessible directory servers.
