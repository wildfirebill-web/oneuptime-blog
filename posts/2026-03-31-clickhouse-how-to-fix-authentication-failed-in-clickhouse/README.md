# How to Fix 'Authentication failed' in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Authentication, Security, Troubleshooting, Access Control

Description: Resolve ClickHouse 'Authentication failed' errors by checking user credentials, access control configuration, and allowed host settings.

---

## Understanding the Error

ClickHouse returns this error when a client provides incorrect credentials or connects from an unauthorized host:

```text
DB::Exception: Authentication failed: password is incorrect, or there is no user with such name.
```

Or with role-based access:

```text
DB::Exception: Authentication failed for user 'analyst'. (AUTHENTICATION_FAILED)
```

## Common Causes

1. Wrong password (including a password that was recently changed)
2. Wrong username (typo or case sensitivity)
3. Connecting from a host not listed in the user's `allowed_hosts`
4. Using SQL-driven access control but the user was created in `users.xml` (or vice versa)
5. Expired or revoked credentials

## Diagnosing the Issue

### Check Which Users Exist

```sql
-- List all users (run as admin)
SELECT name, storage, host_ip, host_names, default_roles_list
FROM system.users
ORDER BY name;
```

### Check Allowed Hosts

```sql
-- See which hosts each user can connect from
SELECT name, host_ip, host_names, host_names_regexp
FROM system.users
WHERE name = 'analyst';
```

### Review Authentication Failures in the Log

```bash
grep -i "authentication failed\|password is incorrect" /var/log/clickhouse-server/clickhouse-server.log | tail -30
```

## Fixing Authentication Issues

### Fix 1 - Reset Password via users.xml

For users defined in `users.xml`:

```xml
<users>
  <analyst>
    <!-- SHA256 of the new password -->
    <password_sha256_hex>...</password_sha256_hex>
    <networks>
      <ip>::/0</ip>
    </networks>
    <profile>default</profile>
    <quota>default</quota>
  </analyst>
</users>
```

Generate the SHA256 hash:

```bash
echo -n "MyNewPassword123" | sha256sum
```

### Fix 2 - Reset Password via SQL (RBAC)

```sql
-- Change password for a SQL-managed user
ALTER USER analyst IDENTIFIED WITH sha256_password BY 'NewSecurePassword!';

-- Verify the change
SHOW CREATE USER analyst;
```

### Fix 3 - Allow Connections from New IP Range

```sql
-- Update allowed hosts via SQL
ALTER USER analyst HOST IP '10.0.0.0/8', '192.168.1.0/24', '127.0.0.1';
```

Or in `users.xml`:

```xml
<analyst>
  <password>MyPassword</password>
  <networks>
    <ip>10.0.0.0/8</ip>
    <ip>192.168.1.0/24</ip>
    <ip>::1</ip>
  </networks>
</analyst>
```

### Fix 4 - Create a Missing User

```sql
-- Create a new user with password and limited access
CREATE USER data_reader
    IDENTIFIED WITH sha256_password BY 'SecurePass123'
    HOST IP '10.0.0.0/8';

-- Grant read access to a database
GRANT SELECT ON analytics.* TO data_reader;
```

### Fix 5 - Debug Connection String

```bash
# Test with explicit credentials from the command line
clickhouse-client \
    --host clickhouse.internal \
    --port 9000 \
    --user analyst \
    --password 'MyPassword' \
    --query "SELECT currentUser()"

# For HTTP interface
curl "http://clickhouse.internal:8123/?user=analyst&password=MyPassword&query=SELECT%201"
```

## Security Best Practices

```sql
-- Use strong passwords (not empty)
ALTER USER default IDENTIFIED WITH sha256_password BY 'ComplexPassword!456';

-- Restrict the default user to localhost only
ALTER USER default HOST LOCAL;

-- Create named users for applications
CREATE USER app_writer
    IDENTIFIED WITH sha256_password BY 'AppWriterPass789'
    HOST IP '10.10.0.0/16';

GRANT INSERT, SELECT ON analytics.* TO app_writer;
```

## Summary

ClickHouse authentication failures are caused by incorrect passwords, IP host restrictions, or mismatches between `users.xml` and SQL-managed users. Diagnose with `system.users` to verify the user exists and check `host_ip` constraints. Fix by resetting the password via `ALTER USER` or `users.xml` and ensuring the client IP is in the allowed hosts list. Always use SHA256-hashed passwords in configuration files to avoid plaintext credential exposure.
