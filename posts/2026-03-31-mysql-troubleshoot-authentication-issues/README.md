# How to Troubleshoot MySQL Authentication Issues

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Authentication, Security, Troubleshooting, User

Description: Diagnose and fix MySQL authentication failures including Access Denied errors, plugin mismatches, and host-based restrictions.

---

## Common Authentication Errors

MySQL authentication failures typically show one of these messages:

- `Access denied for user 'app'@'192.168.1.10' (using password: YES)`
- `Plugin 'caching_sha2_password' could not be loaded`
- `Client does not support authentication protocol requested by server`

## Step 1: Verify the User Account Exists

```sql
-- List all accounts and their authentication plugin
SELECT user, host, plugin, account_locked, password_expired
FROM mysql.user
WHERE user = 'app';
```

MySQL matches accounts using both the username AND the host. An account `'app'@'localhost'` will NOT match a connection from `192.168.1.10` - you need a separate `'app'@'192.168.1.10'` or `'app'@'%'` entry.

## Step 2: Check the Authentication Plugin

MySQL 8 defaults to `caching_sha2_password`. Older clients may only support `mysql_native_password`.

```sql
-- Check which plugin a user is using
SELECT user, host, plugin FROM mysql.user WHERE user = 'app';

-- Change to native password for compatibility with older clients
ALTER USER 'app'@'%' IDENTIFIED WITH mysql_native_password BY 'new_password';
FLUSH PRIVILEGES;
```

For new deployments, prefer keeping `caching_sha2_password` and upgrading the client instead.

## Step 3: Reset a Forgotten Password

```bash
# Stop MySQL and start without grant table checking
sudo systemctl stop mysql
sudo mysqld_safe --skip-grant-tables &

# Connect without a password
mysql -u root
```

```sql
-- Reset the password
FLUSH PRIVILEGES;
ALTER USER 'root'@'localhost' IDENTIFIED BY 'new_strong_password';
FLUSH PRIVILEGES;
EXIT;
```

```bash
# Restart MySQL normally
sudo pkill mysqld_safe
sudo systemctl start mysql
```

## Step 4: Check Account Locking and Password Expiry

```sql
-- Unlock a locked account
ALTER USER 'app'@'%' ACCOUNT UNLOCK;

-- Reset an expired password
ALTER USER 'app'@'%' IDENTIFIED BY 'new_password' PASSWORD EXPIRE NEVER;
```

Check global password expiry policy:

```sql
SHOW VARIABLES LIKE 'default_password_lifetime';
-- Set to 0 to disable automatic expiry
SET GLOBAL default_password_lifetime = 0;
```

## Step 5: Verify Host-Based Access

MySQL uses the most specific matching rule. Check what grants exist for a user:

```sql
SHOW GRANTS FOR 'app'@'%';
SHOW GRANTS FOR 'app'@'localhost';
```

Grant access from a specific subnet:

```sql
-- Allow connections from any host in the 10.0.0.x range
CREATE USER 'app'@'10.0.0.%' IDENTIFIED BY 'password';
GRANT SELECT, INSERT, UPDATE, DELETE ON mydb.* TO 'app'@'10.0.0.%';
FLUSH PRIVILEGES;
```

## Step 6: Check the Error Log

The MySQL error log provides more detail than the client message:

```bash
sudo tail -100 /var/log/mysql/error.log | grep -i "access denied\|auth\|plugin"
```

## Step 7: Enable General Query Log for Debugging

```sql
-- Temporarily enable general log to capture all connection attempts
SET GLOBAL general_log = ON;
SET GLOBAL general_log_file = '/var/log/mysql/general.log';
```

Inspect the log for failed connections, then disable it after debugging (it significantly impacts performance).

```sql
SET GLOBAL general_log = OFF;
```

## Summary

MySQL authentication issues are usually caused by missing host entries, authentication plugin mismatches between the server and client, or expired/locked accounts. Check `mysql.user` to confirm the account and host combination exists, verify the plugin version matches your client, and use the error log to get exact failure reasons. Always use `FLUSH PRIVILEGES` after manual grant table changes.
