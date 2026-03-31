# How to Use GRANT OPTION in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, User, Privilege, Security, Database Administration

Description: Learn how MySQL's WITH GRANT OPTION lets a user delegate their own privileges to others, and when to use or avoid it for security reasons.

---

## What Is GRANT OPTION?

`WITH GRANT OPTION` is an optional clause on a MySQL GRANT statement that allows the recipient to grant the same privilege(s) to other users. Without it, a user can use their privileges but cannot share them. With it, they become a privilege administrator for that scope.

## Granting Privileges WITH GRANT OPTION

```sql
-- alice can now grant SELECT on mydb.* to others
GRANT SELECT ON mydb.* TO 'alice'@'localhost' WITH GRANT OPTION;
```

Alice can now run:

```sql
GRANT SELECT ON mydb.* TO 'carol'@'%';
```

But she cannot grant any privilege she does not herself hold, and she cannot grant privileges beyond the scope she received (`mydb.*`).

## Verifying GRANT OPTION in SHOW GRANTS

```sql
SHOW GRANTS FOR 'alice'@'localhost';
```

Output:

```text
+------------------------------------------------------------------+
| Grants for alice@localhost                                        |
+------------------------------------------------------------------+
| GRANT USAGE ON *.* TO `alice`@`localhost`                         |
| GRANT SELECT ON `mydb`.* TO `alice`@`localhost` WITH GRANT OPTION |
+------------------------------------------------------------------+
```

The `WITH GRANT OPTION` at the end of the line indicates the delegation ability.

## GRANT OPTION at Different Scopes

```sql
-- Global grant option (very powerful - allows granting to any db)
GRANT ALL PRIVILEGES ON *.* TO 'super_dba'@'localhost' WITH GRANT OPTION;

-- Database-scope grant option
GRANT ALL PRIVILEGES ON appdb.* TO 'app_admin'@'%' WITH GRANT OPTION;

-- Table-scope grant option
GRANT SELECT ON mydb.orders TO 'report_manager'@'%' WITH GRANT OPTION;
```

## Revoking GRANT OPTION Without Revoking the Privilege

```sql
REVOKE GRANT OPTION ON mydb.* FROM 'alice'@'localhost';
```

Alice retains SELECT on `mydb.*` but can no longer delegate it to others.

## Revoking All Grants Including GRANT OPTION

```sql
REVOKE ALL PRIVILEGES, GRANT OPTION FROM 'alice'@'localhost';
```

This removes all privileges and the ability to grant them.

## The GRANT OPTION Privilege in information_schema

```sql
SELECT GRANTEE, PRIVILEGE_TYPE, IS_GRANTABLE
FROM information_schema.SCHEMA_PRIVILEGES
WHERE TABLE_SCHEMA = 'mydb'
  AND IS_GRANTABLE = 'YES';
```

`IS_GRANTABLE = 'YES'` means the user has the `GRANT OPTION` for that privilege.

## Security Considerations

`WITH GRANT OPTION` creates a privilege escalation risk. A user with this capability can:

1. Grant their privileges to a new account they create
2. Grant to a less-trusted user or service

Best practices:
- Grant `WITH GRANT OPTION` only to trusted database administrators
- Avoid combining global `ALL PRIVILEGES ... WITH GRANT OPTION` (this is equivalent to a super-user with delegation)
- Audit which users have `WITH GRANT OPTION` regularly

```sql
-- Audit: find all accounts that can delegate privileges
SELECT GRANTEE, PRIVILEGE_TYPE, TABLE_SCHEMA, IS_GRANTABLE
FROM information_schema.SCHEMA_PRIVILEGES
WHERE IS_GRANTABLE = 'YES'
UNION ALL
SELECT GRANTEE, PRIVILEGE_TYPE, '*' AS TABLE_SCHEMA, IS_GRANTABLE
FROM information_schema.USER_PRIVILEGES
WHERE IS_GRANTABLE = 'YES'
ORDER BY GRANTEE;
```

## Roles and GRANT OPTION

For roles, the equivalent is `WITH ADMIN OPTION`:

```sql
GRANT 'developer' TO 'alice'@'localhost' WITH ADMIN OPTION;
```

Alice can now grant the `developer` role to other users.

## Summary

`WITH GRANT OPTION` allows a MySQL user to delegate the same privileges they hold to other accounts. Use it sparingly - only for database administrators or lead developers who need to manage team access. Regularly audit who has `WITH GRANT OPTION` using `information_schema.SCHEMA_PRIVILEGES` and `USER_PRIVILEGES`, and revoke delegation rights when they are no longer needed.
