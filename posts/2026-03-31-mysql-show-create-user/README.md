# How to Use SHOW CREATE USER in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, User, Security, Administration

Description: Learn how to use SHOW CREATE USER in MySQL to view a user account's full creation statement, including authentication and password policy settings.

---

## What Is SHOW CREATE USER?

The `SHOW CREATE USER` statement returns the `CREATE USER` statement that would recreate a given user account, including authentication plugin, password expiry, account locking status, and resource limits. It is the most complete way to inspect a user account definition and is essential for auditing and account migration.

## Basic Syntax

```sql
SHOW CREATE USER 'username'@'host';
SHOW CREATE USER CURRENT_USER();
```

## Viewing the Current User

```sql
SHOW CREATE USER CURRENT_USER()\G
```

Sample output:

```text
*************************** 1. row ***************************
CREATE USER for root@localhost: CREATE USER `root`@`localhost`
  IDENTIFIED WITH `caching_sha2_password`
  AS '$A$005$...(hashed password)...'
  REQUIRE NONE
  PASSWORD EXPIRE DEFAULT
  ACCOUNT UNLOCK
  PASSWORD HISTORY DEFAULT
  PASSWORD REUSE INTERVAL DEFAULT
  PASSWORD REQUIRE CURRENT DEFAULT
  FAILED_LOGIN_ATTEMPTS 0
  PASSWORD_LOCK_TIME 0
```

## Viewing a Specific User

```sql
SHOW CREATE USER 'app_user'@'%'\G
```

```text
CREATE USER `app_user`@`%`
  IDENTIFIED WITH `caching_sha2_password` AS '...'
  REQUIRE SSL
  PASSWORD EXPIRE INTERVAL 90 DAY
  ACCOUNT UNLOCK
  WITH MAX_QUERIES_PER_HOUR 1000
       MAX_CONNECTIONS_PER_HOUR 100
       MAX_USER_CONNECTIONS 20
```

## Creating a User with Full Options

```sql
CREATE USER 'reporting_user'@'10.0.0.%'
  IDENTIFIED BY 'StrongPass!2026'
  REQUIRE SSL
  PASSWORD EXPIRE INTERVAL 90 DAY
  PASSWORD HISTORY 5
  ACCOUNT UNLOCK
  WITH MAX_QUERIES_PER_HOUR 5000
       MAX_USER_CONNECTIONS 10;
```

Then verify what was created:

```sql
SHOW CREATE USER 'reporting_user'@'10.0.0.%'\G
```

## Comparing SHOW CREATE USER vs SHOW GRANTS

These two statements serve different purposes:

```sql
-- SHOW CREATE USER: shows how the account itself is defined
SHOW CREATE USER 'app_user'@'localhost';

-- SHOW GRANTS: shows what privileges the account has
SHOW GRANTS FOR 'app_user'@'localhost';
```

Use both when auditing a user account.

## Listing All Users

```sql
-- See all accounts on the server
SELECT User, Host, plugin, account_locked, password_expired
FROM mysql.user
ORDER BY User, Host;
```

## Migrating a User Account

To replicate a user account on another server:

1. Run `SHOW CREATE USER` on the source and copy the output.
2. On the target, run the output statement.
3. Re-apply grants with `SHOW GRANTS FOR`.

```sql
-- On target server
CREATE USER `app_user`@`%`
  IDENTIFIED WITH `caching_sha2_password` AS '...'
  REQUIRE SSL
  PASSWORD EXPIRE INTERVAL 90 DAY;

GRANT SELECT, INSERT, UPDATE ON mydb.* TO 'app_user'@'%';
```

## Required Privileges

```sql
-- Only superusers or users with CREATE USER privilege can view other accounts
GRANT CREATE USER ON *.* TO 'dba_user'@'localhost';
```

## Summary

`SHOW CREATE USER` is the definitive command for inspecting a MySQL user account's full configuration - authentication plugin, TLS requirements, password policy, and connection limits. Use it together with `SHOW GRANTS` to fully document and replicate user accounts across environments, ensuring security policies are consistently applied.
