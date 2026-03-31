# How to Set Up ClickHouse Access Control with LDAP Groups

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, LDAP, Access Control, Security, Role, Authentication

Description: Learn how to map LDAP directory groups to ClickHouse roles for centralized access control using your organization's existing directory service.

---

Mapping LDAP groups to ClickHouse roles lets you manage access centrally in your directory service. When a user's LDAP group membership changes, their ClickHouse permissions update automatically without manual role reassignment.

## Prerequisites

- LDAP server configured (Active Directory or OpenLDAP)
- ClickHouse LDAP server connection defined in `config.xml`
- SQL-driven access control enabled

## Configuring LDAP Server in config.xml

Define your LDAP server connection:

```text
<ldap_servers>
  <corp_ldap>
    <host>ldap.example.com</host>
    <port>636</port>
    <bind_dn>CN=svc-clickhouse,OU=ServiceAccounts,DC=example,DC=com</bind_dn>
    <bind_password>ServiceAccountPass!</bind_password>
    <user_dn_detection>
      <base_dn>OU=Users,DC=example,DC=com</base_dn>
      <search_filter>(&amp;(objectClass=user)(sAMAccountName={user_name}))</search_filter>
    </user_dn_detection>
    <enable_tls>yes</enable_tls>
    <tls_ca_cert_file>/etc/ssl/certs/ca-certificates.crt</tls_ca_cert_file>
  </corp_ldap>
</ldap_servers>
```

## Creating ClickHouse Roles for LDAP Groups

Create roles that will map to LDAP groups:

```sql
-- Maps to CN=ClickHouse-Analysts,OU=Groups,DC=example,DC=com
CREATE ROLE clickhouse_analysts;
GRANT SELECT ON reporting_db.* TO clickhouse_analysts;

-- Maps to CN=ClickHouse-Engineers,OU=Groups,DC=example,DC=com
CREATE ROLE clickhouse_engineers;
GRANT SELECT, INSERT ON staging_db.* TO clickhouse_engineers;
GRANT SELECT, INSERT ON raw_db.* TO clickhouse_engineers;

-- Maps to CN=ClickHouse-Admins,OU=Groups,DC=example,DC=com
CREATE ROLE clickhouse_admins;
GRANT ALL ON *.* TO clickhouse_admins;
```

## Configuring LDAP-to-Role Mapping in users.xml

Set up an LDAP-authenticated user profile that maps groups to roles:

```text
<users>
  <ldap_user>
    <ldap_server>corp_ldap</ldap_server>
    <ldap_roles>
      <role>
        <base_dn>OU=Groups,DC=example,DC=com</base_dn>
        <search_filter>
          (&amp;(objectClass=group)(member={bind_dn}))
        </search_filter>
        <attribute>cn</attribute>
        <prefix>clickhouse_</prefix>
      </role>
    </ldap_roles>
  </ldap_user>
</users>
```

The `prefix` field strips `clickhouse_` from the LDAP group name to match the ClickHouse role name. For example, group `clickhouse_analysts` maps to role `analysts`.

## Testing Group Mapping

Connect as an LDAP user and verify role assignment:

```sql
-- After logging in as an LDAP user
SELECT currentUser();
-- Returns: jdoe

SHOW CURRENT ROLES;
-- Returns the roles mapped from their LDAP groups
```

Check the system tables to verify:

```sql
SELECT user_name, granted_role_name, with_admin_option
FROM system.role_grants
WHERE user_name = 'jdoe';
```

## Refreshing Group Membership

LDAP group membership is evaluated at login time. If a user's groups change in LDAP, they need to reconnect to pick up new permissions. For long-running connections, set `ldap_roles_cache_ttl_min`:

```text
<ldap_servers>
  <corp_ldap>
    ...
    <roles_cache_ttl_min>5</roles_cache_ttl_min>
  </corp_ldap>
</ldap_servers>
```

## Handling Multiple Group Sources

You can define multiple role mappings from different base DNs:

```text
<ldap_roles>
  <role>
    <base_dn>OU=ClickHouseGroups,DC=example,DC=com</base_dn>
    <search_filter>(&amp;(objectClass=group)(member={bind_dn}))</search_filter>
    <attribute>cn</attribute>
  </role>
  <role>
    <base_dn>OU=SharedGroups,DC=example,DC=com</base_dn>
    <search_filter>(&amp;(objectClass=group)(member={bind_dn}))</search_filter>
    <attribute>cn</attribute>
    <prefix>shared_</prefix>
  </role>
</ldap_roles>
```

## Best Practices

- Use a dedicated service account for ClickHouse LDAP bind with minimal LDAP permissions
- Use group name prefixes to avoid role name collisions
- Pre-create all expected roles in ClickHouse before LDAP users log in
- Monitor authentication errors in `system.query_log` for LDAP failures

```sql
SELECT user, exception, event_time
FROM system.query_log
WHERE exception LIKE '%LDAP%'
ORDER BY event_time DESC
LIMIT 10;
```

## Summary

LDAP group mapping in ClickHouse enables centralized access governance. Configure your LDAP server in `config.xml`, create matching ClickHouse roles, and define group-to-role mapping rules. Changes in LDAP group membership automatically propagate to ClickHouse permissions at the next user login.
