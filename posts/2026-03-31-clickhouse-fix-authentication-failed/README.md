# How to Fix 'Authentication failed' in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Authentication, Security, Error, Troubleshooting

Description: Diagnose and fix 'Authentication failed' errors in ClickHouse by reviewing user credentials, network ACLs, and authentication method settings.

---

ClickHouse "Authentication failed" errors block client connections when credentials are wrong, the user does not exist, or network ACL restrictions apply. The error is deliberately vague for security reasons.

## Verify User Exists

```sql
-- Connect as admin and check users
SELECT name, auth_type, host_ip, host_names
FROM system.users
WHERE name = 'my_user';
```

If the user does not appear, create it:

```sql
CREATE USER my_user IDENTIFIED BY 'secure_password'
HOST ANY;
```

## Reset a Password

```sql
ALTER USER my_user IDENTIFIED BY 'new_secure_password';
```

For users defined in `users.xml` (legacy config), update the password hash:

```bash
# Generate SHA256 hash for users.xml
echo -n 'new_password' | sha256sum
```

```xml
<users>
  <my_user>
    <password_sha256_hex>hash_value_here</password_sha256_hex>
  </my_user>
</users>
```

## Check Host Restrictions

ClickHouse supports IP-based access control per user:

```sql
-- Check current host restrictions
SELECT name, host_ip, host_names
FROM system.users
WHERE name = 'my_user';

-- Allow connections from specific IP
ALTER USER my_user HOST IP '10.0.0.0/8';

-- Allow all hosts (development only)
ALTER USER my_user HOST ANY;
```

## Test with clickhouse-client

```bash
clickhouse-client \
  --host 127.0.0.1 \
  --port 9000 \
  --user my_user \
  --password 'my_password' \
  --query "SELECT 1"
```

## Check Authentication Type

ClickHouse supports multiple auth types (plaintext, SHA256, double SHA1, LDAP, Kerberos):

```sql
SELECT name, auth_type
FROM system.users;
```

For password-based auth, ensure the client and server use matching hash types. Modern ClickHouse defaults to `sha256_password`:

```sql
ALTER USER my_user IDENTIFIED WITH sha256_password BY 'secure_password';
```

## Check SSL Client Certificate Auth

If SSL client certificate auth is configured:

```xml
<!-- users.xml -->
<my_user>
  <ssl_certificates>
    <common_name>client.example.com</common_name>
  </ssl_certificates>
</my_user>
```

Ensure the client presents a valid certificate with the matching CN.

## Review Access Log

```bash
sudo grep "Authentication failed" /var/log/clickhouse-server/clickhouse-server.log | tail -20
```

The log includes the username and source IP for failed attempts.

## Summary

ClickHouse "Authentication failed" errors are caused by wrong passwords, non-existent users, or host ACL mismatches. Use `system.users` to verify the user and its restrictions, reset passwords with `ALTER USER`, check host IP allowlists, and review the server log for source IPs. Prefer SHA256 password hashing over plaintext for production users.
