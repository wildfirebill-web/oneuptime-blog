# How to Query INFORMATION_SCHEMA.SCHEMA_PRIVILEGES in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, INFORMATION_SCHEMA, Security, Privilege, Schema

Description: Learn how to query INFORMATION_SCHEMA.SCHEMA_PRIVILEGES in MySQL to audit database-level permissions granted to user accounts.

---

## What Is INFORMATION_SCHEMA.SCHEMA_PRIVILEGES?

The `INFORMATION_SCHEMA.SCHEMA_PRIVILEGES` table records privileges that have been granted to MySQL users at the database (schema) level. Unlike `USER_PRIVILEGES`, which covers global grants, `SCHEMA_PRIVILEGES` captures permissions scoped to a specific database. These rows correspond to entries in the `mysql.db` system table.

This view is essential when you need to know exactly which users have access to which databases, and what operations they are permitted to perform.

## Columns in SCHEMA_PRIVILEGES

- `GRANTEE` - the user in `'user'@'host'` format
- `TABLE_CATALOG` - always `def`
- `TABLE_SCHEMA` - the database the privilege applies to
- `PRIVILEGE_TYPE` - the privilege name (e.g., `SELECT`, `INSERT`, `CREATE`)
- `IS_GRANTABLE` - `YES` if the privilege was granted with `WITH GRANT OPTION`

## List All Schema-Level Privileges

```sql
SELECT
    GRANTEE,
    TABLE_SCHEMA,
    PRIVILEGE_TYPE,
    IS_GRANTABLE
FROM INFORMATION_SCHEMA.SCHEMA_PRIVILEGES
ORDER BY GRANTEE, TABLE_SCHEMA, PRIVILEGE_TYPE;
```

## Find Who Has Access to a Specific Database

```sql
SELECT
    GRANTEE,
    PRIVILEGE_TYPE,
    IS_GRANTABLE
FROM INFORMATION_SCHEMA.SCHEMA_PRIVILEGES
WHERE TABLE_SCHEMA = 'production_db'
ORDER BY GRANTEE;
```

## Find All Databases a User Can Access

```sql
SELECT
    TABLE_SCHEMA,
    PRIVILEGE_TYPE
FROM INFORMATION_SCHEMA.SCHEMA_PRIVILEGES
WHERE GRANTEE = "'appuser'@'%'"
ORDER BY TABLE_SCHEMA;
```

## Identify Write Access Grants

```sql
SELECT
    GRANTEE,
    TABLE_SCHEMA,
    PRIVILEGE_TYPE
FROM INFORMATION_SCHEMA.SCHEMA_PRIVILEGES
WHERE PRIVILEGE_TYPE IN ('INSERT', 'UPDATE', 'DELETE', 'DROP', 'CREATE')
ORDER BY TABLE_SCHEMA, GRANTEE;
```

## Count Users Per Database

```sql
SELECT
    TABLE_SCHEMA,
    COUNT(DISTINCT GRANTEE) AS user_count
FROM INFORMATION_SCHEMA.SCHEMA_PRIVILEGES
GROUP BY TABLE_SCHEMA
ORDER BY user_count DESC;
```

## Generate REVOKE Statements

To revoke overly broad grants from a user, generate the SQL automatically:

```sql
SELECT
    CONCAT(
        'REVOKE ', PRIVILEGE_TYPE,
        ' ON `', TABLE_SCHEMA, '`.* ',
        'FROM ', GRANTEE, ';'
    ) AS revoke_statement
FROM INFORMATION_SCHEMA.SCHEMA_PRIVILEGES
WHERE GRANTEE = "'olduser'@'localhost'";
```

## Difference from GRANT Syntax

When you run:

```sql
GRANT SELECT, INSERT ON mydb.* TO 'reader'@'localhost';
```

MySQL stores each privilege as a separate row in the underlying `mysql.db` table, which is then surfaced as separate rows in `SCHEMA_PRIVILEGES`. One `GRANT` statement with multiple privileges creates multiple rows in this view.

## Summary

`INFORMATION_SCHEMA.SCHEMA_PRIVILEGES` provides database-scoped access details that are critical for security audits. Query it regularly to identify which users can read or modify each database, detect privilege creep, and generate cleanup scripts that enforce least-privilege policies across your MySQL environment.
