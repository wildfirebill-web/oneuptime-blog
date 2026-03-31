# How to Create a User in ClickHouse with SQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, Security, User Management, CREATE USER

Description: Learn how to create users in ClickHouse with SQL using CREATE USER, set passwords, restrict hosts, assign default roles, and apply settings constraints.

---

ClickHouse supports full SQL-based user management through its Access Control system, introduced in version 20.4. Users can be created, modified, and dropped entirely through SQL statements without editing XML configuration files. SQL-managed users are stored in a persistent access control directory and survive server restarts. This post covers `CREATE USER` syntax, password authentication options, host restrictions, default roles, and settings profiles - everything needed to provision secure database users in ClickHouse.

## Prerequisites - Enabling SQL Access Control

SQL-based access control requires at least one user with management privileges. In a fresh installation, enable it for the `default` user in `users.xml`:

```xml
<users>
  <default>
    <access_management>1</access_management>
    <named_collection_control>1</named_collection_control>
  </default>
</users>
```

After this, all subsequent users can be managed entirely via SQL.

## Basic CREATE USER Syntax

```sql
CREATE USER [IF NOT EXISTS] name
[ON CLUSTER cluster_name]
[IDENTIFIED BY 'password' | IDENTIFIED WITH auth_type BY 'value']
[HOST {LOCAL | NAME 'hostname' | REGEXP 'pattern' | IP 'cidr' | ANY | NONE}]
[DEFAULT ROLE role1, role2, ...]
[DEFAULT DATABASE database]
[SETTINGS setting = value [MIN min] [MAX max] [READONLY | WRITABLE], ...]
[GRANTEES {user | role} [, ...] [EXCEPT {user | role} [, ...]]]
```

A minimal user with a plaintext password:

```sql
CREATE USER IF NOT EXISTS alice IDENTIFIED BY 'str0ng_p4ssword!';
```

## Authentication Methods

### Plaintext Password (SHA256 stored)

```sql
CREATE USER bob IDENTIFIED BY 'my_password';
```

ClickHouse stores the SHA256 hash, not the plaintext.

### SHA256 Password Hash

Pre-hash the password and pass the hex digest to avoid sending plaintext over the wire in migration scripts:

```bash
echo -n 'my_password' | sha256sum | cut -d' ' -f1
```

```sql
CREATE USER bob
IDENTIFIED WITH sha256_hash
BY 'b14a7b8059d9c055954c92674ce60032d1b8f9d2a8a1d8d3f9e16a7f3b4c5d6e';
```

### SHA256 with Salt

```sql
CREATE USER carol
IDENTIFIED WITH sha256_hash
BY 'abc123hashvalue...'
SALT 'random_salt_string';
```

### Double SHA1 (MySQL compatibility)

```sql
CREATE USER legacy_app
IDENTIFIED WITH double_sha1_password BY 'legacy_password';
```

### Bcrypt

```sql
CREATE USER secure_user
IDENTIFIED WITH bcrypt_password BY 'very_secure_password';
```

### No Password (trust-based, local only)

```sql
CREATE USER monitoring_agent IDENTIFIED WITH no_password;
```

Only use no-password authentication with strict host restrictions.

### LDAP Authentication

```sql
CREATE USER ldap_user
IDENTIFIED WITH ldap SERVER 'my_ldap_server';
```

The LDAP server must be configured in `config.xml`.

## Host Restrictions

By default, users can connect from any host. Restrict access to specific IPs, CIDRs, or hostnames:

```sql
-- Only from localhost
CREATE USER local_admin IDENTIFIED BY 'password' HOST LOCAL;

-- Specific IP
CREATE USER app_server IDENTIFIED BY 'password' HOST IP '10.0.1.50';

-- CIDR range
CREATE USER internal_app IDENTIFIED BY 'password' HOST IP '10.0.0.0/8';

-- Hostname
CREATE USER office_user IDENTIFIED BY 'password' HOST NAME 'workstation.example.com';

-- Hostname regex pattern
CREATE USER vpn_user IDENTIFIED BY 'password' HOST REGEXP '.*\.corp\.example\.com';

-- Multiple constraints
CREATE USER restricted IDENTIFIED BY 'password'
HOST IP '10.0.1.0/24', IP '192.168.1.0/24', NAME 'trusted-host.internal';

-- Block all connections (account disabled)
CREATE USER disabled_user IDENTIFIED BY 'password' HOST NONE;
```

## Default Role and Default Database

Assign roles that are automatically activated on login:

```sql
CREATE ROLE analyst;
CREATE ROLE reader;

CREATE USER diana
IDENTIFIED BY 'password'
DEFAULT ROLE analyst, reader
DEFAULT DATABASE analytics;
```

`DEFAULT DATABASE` sets the database used when the user does not specify one in queries or connection strings.

## Settings Constraints

Restrict resource usage by bounding or locking ClickHouse settings for a user:

```sql
CREATE USER etl_job
IDENTIFIED BY 'etl_password'
SETTINGS
    max_memory_usage = 4294967296,                    -- 4 GB hard cap
    max_execution_time = 300 READONLY,                -- cannot exceed 300s, user cannot change
    max_threads = 8 MAX 16,                           -- user can set up to 16 threads
    readonly = 0;                                     -- user can write
```

Setting modifiers:

- `READONLY` - user cannot change this setting
- `MIN value` - user cannot set below this value
- `MAX value` - user cannot set above this value
- `WRITABLE` - explicitly allow user to change the setting (default for most settings)

## Altering a User

```sql
-- Change password
ALTER USER alice IDENTIFIED BY 'new_password';

-- Add host restriction
ALTER USER alice HOST IP '10.0.0.0/8';

-- Add a default role
ALTER USER alice DEFAULT ROLE analyst;

-- Reset to no host restrictions
ALTER USER alice HOST ANY;
```

## Granting Privileges

Users start with no privileges. Grant access after creation:

```sql
-- Grant SELECT on a specific table
GRANT SELECT ON analytics.events TO alice;

-- Grant all privileges on a database
GRANT ALL ON analytics.* TO bob;

-- Grant a role
GRANT analyst TO carol;
```

## Inspecting Users

```sql
-- List all users
SHOW USERS;

-- Detailed information
SELECT
    name,
    auth_type,
    host_ip,
    host_names,
    default_roles_list,
    default_database
FROM system.users;

-- Show grants for a user
SHOW GRANTS FOR alice;

-- Show current user context
SELECT currentUser(), currentDatabase();
```

## Dropping a User

```sql
DROP USER IF EXISTS alice;
```

Dropping a user does not cascade-drop their grants; other users who inherited grants via role chains are unaffected.

## Summary

`CREATE USER` in ClickHouse provides fine-grained control over authentication method, host allowlists, default roles and database, and settings constraints - all through SQL without touching XML files. Start with SHA256 password authentication and CIDR host restrictions for application users, assign default roles at creation time, and use settings constraints to prevent runaway queries from resource-hungry accounts.
