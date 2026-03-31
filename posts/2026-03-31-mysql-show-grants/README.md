# How to Use SHOW GRANTS in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Security, Privilege

Description: Learn how to use SHOW GRANTS in MySQL to view user privileges, role assignments, and column-level permissions for security auditing.

---

## What Is SHOW GRANTS

`SHOW GRANTS` displays the privilege statements that would be needed to recreate the access rights for a MySQL user account. The output is formatted as `GRANT` statements, showing global privileges, database-level privileges, table-level privileges, and role assignments. It is the primary tool for auditing user access in MySQL.

```sql
SHOW GRANTS;                              -- current user
SHOW GRANTS FOR 'username'@'host';        -- specific user
SHOW GRANTS FOR CURRENT_USER();           -- explicit current user
```

## Basic Usage

```sql
-- View your own privileges
SHOW GRANTS;

-- View privileges for a specific user
SHOW GRANTS FOR 'app_user'@'%';
```

```text
+----------------------------------------------------------+
| Grants for app_user@%                                    |
+----------------------------------------------------------+
| GRANT USAGE ON *.* TO `app_user`@`%`                    |
| GRANT SELECT, INSERT, UPDATE ON `myapp_db`.* TO `app_user`@`%` |
| GRANT `app_reader`@`%` TO `app_user`@`%`                |
+----------------------------------------------------------+
```

Each line is a self-contained `GRANT` statement. `GRANT USAGE` is the baseline - it means "user exists" with no specific global privileges.

## Showing Privileges Including Role Grants

When a user has roles assigned, they appear in the output:

```sql
SHOW GRANTS FOR 'analyst'@'%';
```

```text
GRANT USAGE ON *.* TO `analyst`@`%`
GRANT `reporting_reader`@`%`,`reporting_writer`@`%` TO `analyst`@`%`
```

To see the privileges inherited through roles:

```sql
SHOW GRANTS FOR 'analyst'@'%' USING 'reporting_reader', 'reporting_writer';
```

This expands the role grants to show the actual underlying privileges.

## Checking Table-Level and Column-Level Grants

```sql
SHOW GRANTS FOR 'restricted_user'@'localhost';
```

```text
GRANT SELECT (`id`, `name`, `email`) ON `myapp_db`.`users` TO `restricted_user`@`localhost`
```

## Auditing All Users

To audit privileges for all users, query `information_schema` or `mysql` system tables:

```sql
-- List all users and their global privileges
SELECT User, Host,
       Select_priv, Insert_priv, Update_priv, Delete_priv,
       Create_priv, Drop_priv, Grant_priv, Super_priv
FROM mysql.user
ORDER BY User, Host;
```

For database-level grants:

```sql
SELECT User, Host, Db, Select_priv, Insert_priv, Update_priv
FROM mysql.db
ORDER BY User, Host, Db;
```

## Finding Users with Dangerous Privileges

Check for users with `SUPER`, `FILE`, or `GRANT OPTION` privileges:

```sql
SELECT User, Host
FROM mysql.user
WHERE Super_priv = 'Y' OR File_priv = 'Y' OR Grant_priv = 'Y';
```

## Showing Grants for a Role

Roles are stored similarly to users. You can view their grants:

```sql
SHOW GRANTS FOR 'app_reader'@'%';
```

```text
GRANT SELECT ON `myapp_db`.* TO `app_reader`@`%`
```

## Comparing Expected vs Actual Grants

A useful security practice is to capture expected grants and compare against actual:

```bash
# Capture grants for a user
mysql -u root -p -e "SHOW GRANTS FOR 'app_user'@'%';" > expected_grants.sql

# Compare periodically
mysql -u root -p -e "SHOW GRANTS FOR 'app_user'@'%';" | diff expected_grants.sql -
```

## Required Privileges to View Others' Grants

```sql
-- Without elevated privileges, you can only see your own grants
-- With SELECT on mysql.* or the global SELECT privilege, you can see all users
GRANT SELECT ON mysql.* TO 'security_auditor'@'localhost';
```

## Summary

`SHOW GRANTS` provides a human-readable view of user and role privileges formatted as `GRANT` statements. Use it to audit individual accounts, expand role-based privileges with `USING`, and verify that users have exactly the access they need. For bulk auditing across all users, query `mysql.user` and `mysql.db` tables directly.
