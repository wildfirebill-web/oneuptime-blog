# How to Set Password Expiration Policies in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Security, Password Policy, User Management, Administration

Description: Learn how to configure password expiration policies in MySQL to enforce periodic password rotation for user accounts globally or per-user.

---

## Why Set Password Expiration?

Password expiration policies force users to change their passwords after a defined period, reducing the risk of compromised credentials being used indefinitely. MySQL supports both global default expiration intervals and per-account overrides.

## Global Password Expiration Setting

Set a global default expiration interval (in days) for all accounts:

```sql
SET GLOBAL default_password_lifetime = 90;
```

Or in `my.cnf` to persist across restarts:

```text
[mysqld]
default_password_lifetime = 90
```

With this setting, all accounts that do not have an explicit per-account policy will have their passwords expire every 90 days.

A value of `0` disables global expiration:

```text
[mysqld]
default_password_lifetime = 0
```

## Checking the Global Setting

```sql
SHOW VARIABLES LIKE 'default_password_lifetime';
```

## Setting Per-Account Password Expiration

Override the global policy for specific users:

```sql
-- Expire every 60 days
ALTER USER 'service_account'@'%' PASSWORD EXPIRE INTERVAL 60 DAY;

-- Never expire (override global setting)
ALTER USER 'admin'@'localhost' PASSWORD EXPIRE NEVER;

-- Use global default
ALTER USER 'regular_user'@'%' PASSWORD EXPIRE DEFAULT;
```

## Creating a User with an Expiration Policy

```sql
CREATE USER 'contractor'@'%'
  IDENTIFIED BY 'TempPassword!'
  PASSWORD EXPIRE INTERVAL 30 DAY;
```

## Manually Expiring a Password

Force a user to change their password on next login:

```sql
ALTER USER 'bob'@'%' PASSWORD EXPIRE;
```

When `bob` connects, MySQL returns:

```text
ERROR 1820 (HY000): You must reset your password using ALTER USER statement before executing this statement.
```

The user must then run:

```sql
ALTER USER USER() IDENTIFIED BY 'NewPassword!';
```

## Password Reuse Policies (MySQL 8.0)

Prevent users from reusing recent passwords:

```sql
-- Prevent reuse of the last 5 passwords
ALTER USER 'alice'@'%' PASSWORD HISTORY 5;

-- Prevent reuse of passwords set within the last 365 days
ALTER USER 'alice'@'%' PASSWORD REUSE INTERVAL 365 DAY;
```

Or set global defaults:

```text
[mysqld]
password_history = 5
password_reuse_interval = 365
```

## Password Failed Login Lockout (MySQL 8.0)

Lock accounts after consecutive failed login attempts:

```sql
ALTER USER 'alice'@'%'
  FAILED_LOGIN_ATTEMPTS 5
  PASSWORD_LOCK_TIME 2;
```

- `FAILED_LOGIN_ATTEMPTS 5` - lock after 5 consecutive failures
- `PASSWORD_LOCK_TIME 2` - lock for 2 days (use `UNBOUNDED` to lock until manually unlocked)

Unlock a locked account:

```sql
ALTER USER 'alice'@'%' ACCOUNT UNLOCK;
```

## Viewing Password Expiration Status

```sql
SELECT
  user,
  host,
  password_expired,
  password_last_changed,
  password_lifetime
FROM mysql.user
WHERE user NOT IN ('mysql.sys', 'mysql.session', 'mysql.infoschema');
```

```text
+-------+-----------+------------------+------------------------+-------------------+
| user  | host      | password_expired | password_last_changed  | password_lifetime |
+-------+-----------+------------------+------------------------+-------------------+
| alice | %         | N                | 2026-01-01 10:00:00    | 90                |
| bob   | %         | Y                | 2025-10-01 10:00:00    | 30                |
+-------+-----------+------------------+------------------------+-------------------+
```

## Summary

MySQL password expiration policies help enforce security compliance by requiring periodic password rotation. Set a global interval with `default_password_lifetime`, override it per-account with `ALTER USER ... PASSWORD EXPIRE INTERVAL N DAY`, and use `PASSWORD EXPIRE NEVER` for service accounts that cannot tolerate forced password changes. In MySQL 8.0, complement expiration with password history and failed login lockout policies.
