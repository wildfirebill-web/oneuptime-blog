# How to Fix ERROR 1396 Operation CREATE USER Failed in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, User, Error, Account, Security

Description: Fix MySQL ERROR 1396 Operation CREATE USER failed by dropping residual user records from mysql.user, flushing privileges, or using DROP USER IF EXISTS first.

---

MySQL ERROR 1396 occurs when you try to create a user that already exists or has leftover records in the grant tables. The error reads: `ERROR 1396 (HY000): Operation CREATE USER failed for 'username'@'host'`.

## Why This Happens

The most common causes are:
- The user was deleted with `DELETE FROM mysql.user` directly instead of `DROP USER`
- A partial `CREATE USER` left inconsistent records in the grant tables
- The user exists in `mysql.user` but not in the `mysql.db` or `mysql.proxies_priv` tables
- Replication transferred a `DROP USER` but not the subsequent `CREATE USER`

## Check If the User Already Exists

```sql
-- Check the mysql.user table
SELECT User, Host, account_locked, password_expired
FROM mysql.user
WHERE User = 'myapp';

-- List all users with this name
SELECT User, Host FROM mysql.user WHERE User LIKE 'myapp%';
```

## Fix 1: Drop the User First

If the user exists, drop them cleanly before recreating:

```sql
-- Use IF EXISTS to avoid an error if they do not exist
DROP USER IF EXISTS 'myapp'@'localhost';
DROP USER IF EXISTS 'myapp'@'%';

-- Now create the user
CREATE USER 'myapp'@'%' IDENTIFIED BY 'strong_password';
GRANT SELECT, INSERT, UPDATE, DELETE ON mydb.* TO 'myapp'@'%';
FLUSH PRIVILEGES;
```

## Fix 2: Clean Up Inconsistent Grant Tables

If the user record exists but DROP USER fails:

```sql
-- Check all grant tables for orphaned records
SELECT * FROM mysql.user WHERE User = 'myapp';
SELECT * FROM mysql.db WHERE User = 'myapp';
SELECT * FROM mysql.tables_priv WHERE User = 'myapp';
SELECT * FROM mysql.columns_priv WHERE User = 'myapp';

-- Delete orphaned records manually (as a last resort)
DELETE FROM mysql.db WHERE User = 'myapp';
DELETE FROM mysql.tables_priv WHERE User = 'myapp';
DELETE FROM mysql.columns_priv WHERE User = 'myapp';
DELETE FROM mysql.user WHERE User = 'myapp' AND Host = '%';

FLUSH PRIVILEGES;
```

After flushing, try `CREATE USER` again.

## Fix 3: Use FLUSH PRIVILEGES

Sometimes the in-memory grant cache is out of sync:

```sql
FLUSH PRIVILEGES;

-- Retry the CREATE USER
CREATE USER 'myapp'@'%' IDENTIFIED BY 'strong_password';
```

## Fix 4: Use CREATE USER ... IF NOT EXISTS

MySQL 5.7.6+ supports `IF NOT EXISTS` to avoid the error:

```sql
CREATE USER IF NOT EXISTS 'myapp'@'%' IDENTIFIED BY 'strong_password';
```

This will not update the password if the user already exists.

## Fix 5: Use CREATE OR REPLACE USER (MySQL 8.0+)

```sql
CREATE OR REPLACE USER 'myapp'@'%' IDENTIFIED BY 'new_password';
GRANT SELECT, INSERT, UPDATE ON mydb.* TO 'myapp'@'%';
```

## Verify After Fix

```sql
-- Confirm the user exists and is correct
SHOW GRANTS FOR 'myapp'@'%';

-- Test the connection from another session
-- mysql -u myapp -p -h hostname
```

## Summary

ERROR 1396 is caused by leftover or inconsistent records in MySQL's grant tables. The cleanest fix is `DROP USER IF EXISTS` followed by `CREATE USER`. For persistent issues, audit all grant tables for orphaned records and clean them with direct DELETE statements followed by `FLUSH PRIVILEGES`. Prefer `DROP USER` over direct `DELETE FROM mysql.user` to keep grant tables consistent.
