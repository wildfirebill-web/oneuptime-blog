# How to Use LDAP Authentication in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, LDAP, Security, Authentication, Database, Configuration

Description: Learn how to configure ClickHouse to authenticate users against an LDAP directory, map LDAP groups to ClickHouse roles, and manage centralized identity.

---

ClickHouse supports LDAP authentication, allowing users to log in with their corporate directory credentials instead of a ClickHouse-specific password. This is useful in organizations that use Active Directory, OpenLDAP, or any LDAPv3-compatible server as their identity provider.

## How LDAP Authentication Works in ClickHouse

ClickHouse does not maintain a local password for LDAP-authenticated users. When a user connects, ClickHouse:

1. Binds to the LDAP server using the provided credentials
2. If the bind succeeds, the user is authenticated
3. ClickHouse applies the user's local privileges (defined by SQL grants or roles)

LDAP is only used for authentication. Authorization (what the user can do) is still managed by ClickHouse's own privilege system, though LDAP group membership can be mapped to ClickHouse roles automatically.

## Configuring the LDAP Server Connection

Add the LDAP server definition to `/etc/clickhouse-server/config.d/ldap.xml`:

```xml
<clickhouse>
  <ldap_servers>
    <corporate_ldap>
      <!-- LDAP server hostname or IP -->
      <host>ldap.corp.example.com</host>
      <!-- 389 for plain LDAP, 636 for LDAPS -->
      <port>636</port>
      <!-- Use ldaps (TLS) or starttls -->
      <enable_tls>yes</enable_tls>
      <tls_minimum_protocol_version>tls1.2</tls_minimum_protocol_version>
      <!-- CA certificate for TLS verification -->
      <tls_ca_cert_file>/etc/ssl/certs/ca-certificates.crt</tls_ca_cert_file>

      <!-- Bind DN template: {user_name} is replaced with the login name -->
      <bind_dn>uid={user_name},ou=people,dc=corp,dc=example,dc=com</bind_dn>

      <!-- Base DN for group search (used for role mapping) -->
      <user_dn_detection>
        <base_dn>ou=people,dc=corp,dc=example,dc=com</base_dn>
        <search_filter>(&amp;(objectClass=posixAccount)(uid={user_name}))</search_filter>
      </user_dn_detection>
    </corporate_ldap>
  </ldap_servers>
</clickhouse>
```

## Configuring an LDAP-Authenticated User

In `/etc/clickhouse-server/users.xml`, define a user that authenticates via LDAP:

```xml
<users>
  <alice>
    <ldap>
      <!-- Reference the LDAP server defined above -->
      <server>corporate_ldap</server>
    </ldap>
    <networks>
      <ip>::/0</ip>
    </networks>
    <profile>default</profile>
    <quota>default</quota>
  </alice>
</users>
```

When `alice` connects to ClickHouse with her LDAP password, ClickHouse verifies it against `corporate_ldap`. No password is stored locally.

## Creating SQL-Managed LDAP Users

Alternatively, create users via SQL without editing `users.xml`:

```sql
CREATE USER alice
    IDENTIFIED WITH ldap SERVER 'corporate_ldap'
    HOST IP '10.0.0.0/8';
```

Grant privileges as normal:

```sql
GRANT SELECT ON analytics.* TO alice;
```

## Active Directory Bind DN Format

Active Directory uses a different DN format (UPN or `sAMAccountName`):

```xml
<corporate_ad>
  <host>ad.corp.example.com</host>
  <port>636</port>
  <enable_tls>yes</enable_tls>
  <!-- UPN format for Active Directory -->
  <bind_dn>{user_name}@corp.example.com</bind_dn>
  <!-- Or use sAMAccountName bind -->
  <!-- <bind_dn>CORP\{user_name}</bind_dn> -->
</corporate_ad>
```

## Mapping LDAP Groups to ClickHouse Roles

ClickHouse can query LDAP for group membership and automatically assign matching ClickHouse roles. Configure `role_mappings` in the LDAP server block:

```xml
<corporate_ldap>
  <host>ldap.corp.example.com</host>
  <port>636</port>
  <enable_tls>yes</enable_tls>
  <bind_dn>uid={user_name},ou=people,dc=corp,dc=example,dc=com</bind_dn>

  <role_mappings>
    <mapping>
      <!-- Base DN for group search -->
      <base_dn>ou=groups,dc=corp,dc=example,dc=com</base_dn>
      <!-- Search filter: find groups where the user is a member -->
      <search_filter>(&amp;(objectClass=groupOfNames)(member={user_dn}))</search_filter>
      <!-- Attribute holding the group name -->
      <attribute>cn</attribute>
      <!-- Prefix to strip from group names before matching ClickHouse roles -->
      <prefix>clickhouse_</prefix>
    </mapping>
  </role_mappings>
</corporate_ldap>
```

With this configuration, if a user belongs to LDAP group `clickhouse_analytics`, ClickHouse assigns the `analytics` role to that user on login (the `clickhouse_` prefix is stripped).

Create the corresponding roles in ClickHouse:

```sql
CREATE ROLE analytics;
GRANT SELECT ON analytics.* TO analytics;

CREATE ROLE admin;
GRANT ALL ON *.* TO admin;
```

## Creating a User with LDAP + Role Mapping

```sql
CREATE USER '{ldap_user}'
    IDENTIFIED WITH ldap SERVER 'corporate_ldap'
    HOST IP '::/0';
```

The `{ldap_user}` is a placeholder name. ClickHouse creates a user entry on first successful login and applies mapped roles.

To allow any LDAP user to authenticate (not just pre-defined ones), use the built-in `allow_ldap_server` option in `users.xml`:

```xml
<users>
  <default>
    <!-- existing default user config -->
  </default>
</users>
```

In newer ClickHouse versions, use `CREATE USER ... IDENTIFIED WITH ldap` combined with user directories (see the ClickHouse documentation on external authenticators).

## Verifying LDAP Authentication

Test the LDAP connection from the ClickHouse host:

```bash
ldapsearch -H ldaps://ldap.corp.example.com \
  -D "uid=alice,ou=people,dc=corp,dc=example,dc=com" \
  -W \
  -b "ou=people,dc=corp,dc=example,dc=com" \
  "(uid=alice)"
```

If the search returns results, the bind DN and credentials are correct.

Connect to ClickHouse as the LDAP user:

```bash
clickhouse-client --host localhost --user alice --password
```

## Troubleshooting

**Authentication failed:**
Check that the bind DN template produces a valid DN for the user. Test the bind manually with `ldapsearch`.

**TLS errors:**
Verify the CA certificate path and that the certificate chain is complete:

```bash
openssl s_client -connect ldap.corp.example.com:636 -CAfile /etc/ssl/certs/ca-certificates.crt
```

**Roles not assigned:**
Confirm the LDAP group search filter returns the expected groups and that the ClickHouse role names match the group names after prefix stripping.

## Summary

ClickHouse LDAP authentication lets users log in with their corporate directory credentials by delegating password verification to an LDAP server. Configure the server connection in `ldap_servers`, create users with `IDENTIFIED WITH ldap`, and optionally map LDAP groups to ClickHouse roles for automatic privilege assignment. Authorization remains ClickHouse-managed through its own privilege system.
