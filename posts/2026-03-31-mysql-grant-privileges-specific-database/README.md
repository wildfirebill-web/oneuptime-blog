# How to Grant Privileges on a Specific Database in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, User, Privilege, Security, Database Administration

Description: Learn how to use MySQL GRANT to assign specific privileges on a single database, limiting a user's access to only the schema they need.

---

## Why Scope Grants to a Database?

In multi-tenant or multi-application MySQL servers, each application typically owns one or more databases. Granting privileges at the database level isolates applications from each other - a bug or compromise in one application cannot affect another application's data.

## Syntax for Database-Level GRANT

```sql
GRANT privilege_list ON database_name.* TO 'user'@'host';
```

The `*` after the database name means all tables in that database, including tables created in the future.

## Common Database-Level Grants

```sql
-- Full application access (read/write, no schema changes)
GRANT SELECT, INSERT, UPDATE, DELETE ON appdb.* TO 'appuser'@'%';

-- Full access including schema changes (for migration tools)
GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, DROP, ALTER, INDEX,
      CREATE VIEW, SHOW VIEW, EXECUTE, CREATE ROUTINE, ALTER ROUTINE,
      EVENT, TRIGGER ON appdb.* TO 'migrator'@'localhost';

-- Read-only access for reporting
GRANT SELECT ON appdb.* TO 'reporter'@'192.168.1.0/255.255.255.0';
```

## Granting on Multiple Databases

Each database requires its own GRANT statement:

```sql
GRANT SELECT, INSERT, UPDATE, DELETE ON db1.* TO 'app'@'%';
GRANT SELECT, INSERT, UPDATE, DELETE ON db2.* TO 'app'@'%';
```

## Listing All Database-Level Privileges for a User

```sql
SHOW GRANTS FOR 'appuser'@'%';
```

Sample output:

```text
+-----------------------------------------------------------------+
| Grants for appuser@%                                           |
+-----------------------------------------------------------------+
| GRANT USAGE ON *.* TO `appuser`@`%`                            |
| GRANT SELECT, INSERT, UPDATE, DELETE ON `appdb`.* TO `appuser`@`%` |
+-----------------------------------------------------------------+
```

## Inspecting All Database Grants via information_schema

```sql
SELECT GRANTEE, TABLE_CATALOG, TABLE_SCHEMA, PRIVILEGE_TYPE, IS_GRANTABLE
FROM information_schema.SCHEMA_PRIVILEGES
WHERE TABLE_SCHEMA = 'appdb'
ORDER BY GRANTEE, PRIVILEGE_TYPE;
```

## Revoking a Specific Privilege on a Database

```sql
-- Remove DELETE privilege but keep others
REVOKE DELETE ON appdb.* FROM 'appuser'@'%';
```

Verify the change:

```sql
SHOW GRANTS FOR 'appuser'@'%';
```

## Granting Future Tables Automatically

When you grant on `database_name.*`, any table created in that database after the grant is automatically covered - no additional GRANT is needed for new tables.

## Example - Setting Up an Application User

```sql
-- Step 1: Create the user
CREATE USER 'webapp'@'%' IDENTIFIED BY 'Str0ng!AppPass2024';

-- Step 2: Grant application privileges on the app database only
GRANT SELECT, INSERT, UPDATE, DELETE ON ecommerce.* TO 'webapp'@'%';

-- Step 3: Verify
SHOW GRANTS FOR 'webapp'@'%';
```

This user can do everything an application needs (CRUD) but cannot access other databases, create or drop tables, or run administrative commands.

## Summary

Database-level GRANTs scope a user's privileges to exactly one schema using the `db_name.*` notation. Grant only the privileges the account actually needs (typically SELECT, INSERT, UPDATE, DELETE for application users) and avoid global `*.*` grants. Pair every GRANT with a verification using `SHOW GRANTS` to confirm the access is exactly as intended.
