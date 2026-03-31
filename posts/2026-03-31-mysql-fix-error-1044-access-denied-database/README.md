# How to Fix ERROR 1044 Access Denied for User to Database in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Error, Privilege, Security, Troubleshooting

Description: Learn how to fix MySQL ERROR 1044 Access denied for user to database by granting the correct database-level privileges.

---

## What Is ERROR 1044?

```text
ERROR 1044 (42000): Access denied for user 'appuser'@'localhost' to database 'mydb'
```

This error occurs when a user tries to access or operate on a database they have not been granted access to. Unlike ERROR 1142 (which is about specific command privileges), ERROR 1044 is about database-level access being denied entirely.

## Common Causes

- The user account exists but has not been granted any privileges on the target database
- The user is connecting with the wrong host (e.g., `'%'` vs `'localhost'`)
- The database name was mistyped in the GRANT statement
- The user was created but no `GRANT` was issued

## Step 1: Connect as Root and Check the User

```sql
SELECT user, host FROM mysql.user WHERE user = 'appuser';
```

Check what grants already exist:

```sql
SHOW GRANTS FOR 'appuser'@'localhost';
```

If the only line is:

```text
GRANT USAGE ON *.* TO 'appuser'@'localhost'
```

The user can log in but has no access to any database.

## Step 2: Verify the Database Exists

```sql
SHOW DATABASES LIKE 'mydb';
```

If the database does not appear, you may need to create it:

```sql
CREATE DATABASE mydb CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

## Step 3: Grant Access to the Database

Grant read/write access:

```sql
GRANT SELECT, INSERT, UPDATE, DELETE ON mydb.* TO 'appuser'@'localhost';
FLUSH PRIVILEGES;
```

Grant full access including schema changes:

```sql
GRANT ALL PRIVILEGES ON mydb.* TO 'appuser'@'localhost';
FLUSH PRIVILEGES;
```

## Step 4: Handle Host Mismatch

The host in the user account must match the host from which the connection is made:

```sql
-- User created with 'localhost' cannot connect from a remote IP
-- Create an additional user with % for remote connections
CREATE USER 'appuser'@'%' IDENTIFIED BY 'securepassword';
GRANT ALL PRIVILEGES ON mydb.* TO 'appuser'@'%';
FLUSH PRIVILEGES;
```

## Step 5: Granting Permission to Create Databases

If the user needs to create databases themselves:

```sql
GRANT CREATE ON *.* TO 'appuser'@'localhost';
FLUSH PRIVILEGES;
```

Or let them create only databases matching a pattern (not supported directly, but you can grant on a specific database):

```sql
GRANT ALL PRIVILEGES ON `appuser_%`.* TO 'appuser'@'localhost';
FLUSH PRIVILEGES;
```

## Step 6: Verify the Fix

Log in as the affected user and check:

```sql
mysql -u appuser -p
SHOW DATABASES;
USE mydb;
```

## Checking All Grants in the System

If you are troubleshooting multiple users:

```sql
SELECT * FROM information_schema.USER_PRIVILEGES;
SELECT * FROM information_schema.SCHEMA_PRIVILEGES WHERE GRANTEE LIKE '%appuser%';
```

## Summary

ERROR 1044 means the user account lacks database-level access. Fix it by running `GRANT ... ON mydb.* TO 'user'@'host'` as a privileged user, followed by `FLUSH PRIVILEGES`. Ensure the host in the grant matches the connection source, and verify the target database exists before granting. Use `SHOW GRANTS FOR 'user'@'host'` to audit current permissions.
