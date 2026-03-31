# How to Set Password Expiration Policies in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Security, Password, Policy, Authentication, User Management

Description: Learn how to configure MySQL password expiration policies globally and per user to enforce regular credential rotation and improve security.

---

## Overview

MySQL supports password expiration at two levels: a global default policy that applies to all accounts, and per-account overrides. When a password expires, the account enters a "sandbox" mode where the user can only change their password until they do so - all other queries fail. This is a key control for regulatory compliance and organizational security policies.

## Global Password Expiration Policy

The global policy is controlled by the `default_password_lifetime` system variable. It sets the number of days after which passwords expire for all accounts that do not have an explicit per-account policy.

### View the Current Global Policy

```sql
SHOW VARIABLES LIKE 'default_password_lifetime';
```

The default value is `0`, which means passwords never expire globally.

### Set the Global Policy in my.cnf

```ini
[mysqld]
default_password_lifetime = 90
```

This sets all passwords to expire after 90 days unless overridden per account.

### Set the Global Policy at Runtime

```sql
SET GLOBAL default_password_lifetime = 90;
```

Note: this change does not persist after a restart unless it is also written to `my.cnf`.

## Per-Account Password Expiration

Override the global policy for individual accounts using `CREATE USER` or `ALTER USER`:

### Expire After N Days

```sql
-- New account with 60-day expiry
CREATE USER 'alice'@'%' IDENTIFIED BY 'StrongPass!' PASSWORD EXPIRE INTERVAL 60 DAY;

-- Update existing account
ALTER USER 'alice'@'%' PASSWORD EXPIRE INTERVAL 60 DAY;
```

### Password Never Expires (Override Global)

```sql
ALTER USER 'svc_account'@'%' PASSWORD EXPIRE NEVER;
```

Service accounts typically should not expire, but use a strong, randomly generated password.

### Expire on Next Login (Force Immediate Change)

```sql
ALTER USER 'new_employee'@'%' PASSWORD EXPIRE;
```

The user must run `ALTER USER ... IDENTIFIED BY '...'` as their first action after logging in.

### Revert to Global Policy

```sql
ALTER USER 'alice'@'%' PASSWORD EXPIRE DEFAULT;
```

## Checking Expiration Status

```sql
SELECT
    user,
    host,
    password_expired,
    password_last_changed,
    password_lifetime
FROM mysql.user
WHERE user NOT LIKE 'mysql.%';
```

- `password_expired = 'Y'` means the password has expired and must be changed.
- `password_lifetime = NULL` means the account follows the global policy.

## What Happens When a Password Expires

When a user connects with an expired password, MySQL puts the session in "expired password" mode:

```
ERROR 1820 (HY000): You must reset your password using ALTER USER statement before executing this statement.
```

The user can only run:

```sql
ALTER USER USER() IDENTIFIED BY 'NewStrongPassword!';
```

All other queries fail until the password is changed.

## Automating Expiry Notifications

Check for accounts expiring within the next 7 days using a scheduled event or monitoring script:

```sql
SELECT
    user,
    host,
    password_last_changed,
    password_lifetime,
    DATE_ADD(password_last_changed, INTERVAL IFNULL(password_lifetime, @@default_password_lifetime) DAY) AS expires_on
FROM mysql.user
WHERE password_lifetime IS NOT NULL
   OR @@default_password_lifetime > 0
HAVING expires_on BETWEEN CURDATE() AND DATE_ADD(CURDATE(), INTERVAL 7 DAY);
```

## Password Reuse Policy

MySQL 8.0 also supports preventing password reuse:

```sql
-- Prevent reusing the last 5 passwords
ALTER USER 'alice'@'%' PASSWORD HISTORY 5;

-- Prevent reuse within 365 days
ALTER USER 'alice'@'%' PASSWORD REUSE INTERVAL 365 DAY;
```

Set globally in `my.cnf`:

```ini
[mysqld]
password_history        = 5
password_reuse_interval = 365
```

## Summary

MySQL password expiration policies protect against stale credentials by forcing periodic password rotation. Set a global policy with `default_password_lifetime` in `my.cnf`, and override it per account with `ALTER USER ... PASSWORD EXPIRE INTERVAL N DAY`. Pair expiration with reuse restrictions (`PASSWORD HISTORY`, `PASSWORD REUSE INTERVAL`) and the `validate_password` component for a complete password security posture.
