# How to Use FLUSH PRIVILEGES in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Security, Administration

Description: Learn when and how to use FLUSH PRIVILEGES in MySQL to reload grant tables and apply user permission changes made directly to system tables.

---

## What Is FLUSH PRIVILEGES

`FLUSH PRIVILEGES` instructs MySQL to re-read the grant tables from the `mysql` system database and update its in-memory privilege cache. MySQL stores user accounts, passwords, and access rights in tables like `mysql.user`, `mysql.db`, and `mysql.tables_priv`. When these tables are modified through `GRANT`, `REVOKE`, `CREATE USER`, or `DROP USER` statements, MySQL automatically refreshes the cache. However, when you modify those tables directly with `INSERT`, `UPDATE`, or `DELETE`, you must run `FLUSH PRIVILEGES` manually.

```sql
FLUSH PRIVILEGES;
```

## When You Need FLUSH PRIVILEGES

You need `FLUSH PRIVILEGES` only when you bypass the standard privilege statements and write directly to the grant tables:

```sql
-- Direct INSERT into mysql.user - requires FLUSH PRIVILEGES after
INSERT INTO mysql.user (Host, User, authentication_string, plugin)
VALUES ('localhost', 'analyst', '', 'caching_sha2_password');

FLUSH PRIVILEGES;
```

You do NOT need `FLUSH PRIVILEGES` when using:

```sql
-- These auto-reload the grant tables
CREATE USER 'analyst'@'localhost' IDENTIFIED BY 'securepass';
GRANT SELECT ON reports.* TO 'analyst'@'localhost';
REVOKE SELECT ON reports.* FROM 'analyst'@'localhost';
DROP USER 'analyst'@'localhost';
```

## Required Privilege

To run `FLUSH PRIVILEGES`, you need the `RELOAD` privilege:

```sql
GRANT RELOAD ON *.* TO 'dba_user'@'localhost';
```

## Example: Creating a User Manually

This pattern is sometimes used in automation scripts or when restoring a database:

```sql
-- Step 1: insert the user record
INSERT INTO mysql.user (
  Host, User, authentication_string, plugin,
  Select_priv, Insert_priv, Update_priv
) VALUES (
  '%', 'app_user', SHA2('MyPassword123', 256), 'caching_sha2_password',
  'Y', 'Y', 'Y'
);

-- Step 2: apply the changes
FLUSH PRIVILEGES;
```

After the flush, `app_user` can connect to MySQL.

## Verifying the Effect

You can verify that privilege changes have been applied by checking the session privileges:

```sql
-- Check current user privileges
SHOW GRANTS FOR 'analyst'@'localhost';

-- Check in the grant tables directly
SELECT User, Host, Select_priv, Insert_priv
FROM mysql.user
WHERE User = 'analyst';
```

## FLUSH PRIVILEGES on a Replica

If you are running replication and you manually modify grant tables on the primary, the `INSERT`/`UPDATE`/`DELETE` statements will be replicated to replicas. However, the `FLUSH PRIVILEGES` command itself will also be replicated (unless you use `FLUSH LOCAL PRIVILEGES`):

```sql
-- Replicate the flush to all replicas
FLUSH PRIVILEGES;

-- Run flush only locally, do not replicate
FLUSH LOCAL PRIVILEGES;
-- or equivalently
FLUSH NO_WRITE_TO_BINLOG PRIVILEGES;
```

## Common Mistake: Unnecessary FLUSH PRIVILEGES

A frequent misconception is to run `FLUSH PRIVILEGES` after every user management operation. This is unnecessary and slightly wasteful:

```sql
-- No need to flush after these
CREATE USER 'reader'@'%' IDENTIFIED BY 'pass';
GRANT SELECT ON app_db.* TO 'reader'@'%';
FLUSH PRIVILEGES; -- unnecessary here
```

Running it when not needed is harmless but adds confusion about when it is actually required.

## Summary

`FLUSH PRIVILEGES` reloads the grant tables into memory and is only necessary when you directly modify the `mysql` system tables using DML statements. Always prefer `CREATE USER`, `GRANT`, `REVOKE`, and `DROP USER` which handle cache updates automatically, reserving `FLUSH PRIVILEGES` for direct table edits in scripts or recovery scenarios.
