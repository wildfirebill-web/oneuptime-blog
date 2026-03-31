# How to Use SHOW PRIVILEGES in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Privilege, Security, Administration

Description: Learn how to use SHOW PRIVILEGES in MySQL to list all supported privilege types, understand their contexts, and apply them in user management.

---

## What Is SHOW PRIVILEGES?

The `SHOW PRIVILEGES` statement in MySQL returns a list of all privilege types that the server supports. Unlike `SHOW GRANTS`, which displays privileges assigned to a specific user, `SHOW PRIVILEGES` shows every privilege that can potentially be granted - regardless of what any user actually has.

This is a useful reference when writing grant statements or auditing which privilege types are available on your MySQL version.

## Basic Syntax

```sql
SHOW PRIVILEGES;
```

No arguments are required. The statement returns a result set with three columns:

- `Privilege` - the privilege name (e.g., `SELECT`, `SUPER`, `REPLICATION SLAVE`)
- `Context` - where the privilege applies (e.g., `Tables`, `Server Admin`, `Databases`)
- `Comment` - a short description of what the privilege allows

## Sample Output

```sql
SHOW PRIVILEGES\G
```

```text
*************************** 1. row ***************************
Privilege: Alter
  Context: Tables
  Comment: To alter the table

*************************** 2. row ***************************
Privilege: Alter Routine
  Context: Functions,Procedures
  Comment: To alter or drop stored functions/procedures

*************************** 3. row ***************************
Privilege: Create
  Context: Databases,Tables,Indexes
  Comment: To create new databases and tables
...
```

## Understanding Privilege Context

Each privilege has a context that defines where it can be granted:

```sql
-- Privileges that apply at the global level
-- Context: Server Admin, Databases, Tables, etc.

-- Grant a table-level privilege
GRANT SELECT ON mydb.mytable TO 'app_user'@'localhost';

-- Grant a database-level privilege
GRANT CREATE ON mydb.* TO 'dev_user'@'%';

-- Grant a global privilege
GRANT REPLICATION SLAVE ON *.* TO 'replica_user'@'10.0.0.2';
```

## Combining with SHOW GRANTS

Use `SHOW PRIVILEGES` to discover available options, then use `SHOW GRANTS` to see what a user actually holds:

```sql
-- See all possible privileges
SHOW PRIVILEGES;

-- See privileges granted to a specific user
SHOW GRANTS FOR 'app_user'@'localhost';

-- See privileges for the current user
SHOW GRANTS;
```

## Filtering Specific Contexts

You can query the `information_schema.USER_PRIVILEGES` view for programmatic access:

```sql
SELECT PRIVILEGE_TYPE, IS_GRANTABLE
FROM information_schema.USER_PRIVILEGES
WHERE GRANTEE = "'app_user'@'localhost'";
```

## Practical Use Cases

Use `SHOW PRIVILEGES` to:

- Verify that a privilege name is valid before including it in a `GRANT` statement
- Understand which privileges exist in a specific MySQL version
- Prepare audit scripts that enumerate all privilege types

```sql
-- Example: grant only what is needed (principle of least privilege)
GRANT SELECT, INSERT, UPDATE ON app_db.* TO 'api_user'@'%';
FLUSH PRIVILEGES;
```

## Summary

`SHOW PRIVILEGES` is a simple but valuable statement for any database administrator or developer working with MySQL security. It provides a complete reference of every privilege the server supports, including the context in which each privilege applies. Pair it with `SHOW GRANTS` and `information_schema` queries to build robust, least-privilege access policies for your MySQL environment.
