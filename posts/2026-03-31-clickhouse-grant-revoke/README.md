# How to Use GRANT and REVOKE in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, Security, GRANT, REVOKE, Access Control

Description: Learn how to manage user and role permissions in ClickHouse using GRANT and REVOKE, covering privilege types, database and table scoping, and role-based access control.

---

ClickHouse includes a full SQL-driven access control system. Privileges are granted to users or roles using GRANT and removed using REVOKE. Roles let you bundle privileges into reusable units, simplifying permission management across many users. This post covers every major privilege type, the scoping options, and practical patterns for production access control.

## Prerequisites - Enable SQL-Driven Access Control

SQL access control must be enabled for the user account that will run GRANT and REVOKE statements. Ensure `access_management = 1` is set for the admin account in `users.xml`, or use the `default` user which has it enabled in newer ClickHouse versions.

## GRANT Syntax

```sql
GRANT privilege [(column_list)] ON {database.table | database.* | *.*} TO {user | role};
```

### Grant SELECT on a Specific Table

```sql
GRANT SELECT ON analytics.events TO alice;
```

### Grant SELECT on All Tables in a Database

```sql
GRANT SELECT ON analytics.* TO alice;
```

### Grant SELECT on All Databases

```sql
GRANT SELECT ON *.* TO alice;
```

## Privilege Types

| Privilege | Scope | Description |
|-----------|-------|-------------|
| SELECT | Table / Database | Read data |
| INSERT | Table / Database | Write data |
| ALTER | Table / Database | ALTER TABLE statements |
| CREATE | Database | CREATE TABLE, VIEW, etc. |
| DROP | Database / Table | DROP TABLE, DATABASE |
| TRUNCATE | Table | TRUNCATE TABLE |
| OPTIMIZE | Table | OPTIMIZE TABLE |
| SHOW | Database / Table | SHOW TABLES, SHOW DATABASES |
| SYSTEM | Global | SYSTEM RELOAD, FLUSH, etc. |
| dictGet | Dictionary | Read from dictionaries |
| ALL | Any | All privileges for the scope |

```sql
-- Grant full access to a database
GRANT ALL ON analytics.* TO alice;

-- Grant only INSERT and SELECT
GRANT SELECT, INSERT ON analytics.events TO bob;

-- Grant dictionary read access
GRANT dictGet ON dim.products_dict TO reporting_user;
```

## Column-Level Privileges

SELECT and INSERT can be scoped to specific columns:

```sql
-- Allow reading only two columns
GRANT SELECT(event_id, event_type) ON analytics.events TO limited_user;

-- Allow inserting only specific columns
GRANT INSERT(event_id, event_type, created_at) ON analytics.events TO ingest_user;
```

## Role-Based Access Control

Creating roles lets you manage permissions for groups of users without repeating GRANT statements.

```sql
-- Create roles
CREATE ROLE analyst;
CREATE ROLE data_engineer;

-- Grant privileges to roles
GRANT SELECT ON analytics.* TO analyst;
GRANT SELECT, INSERT, CREATE, DROP ON staging.* TO data_engineer;

-- Assign roles to users
GRANT analyst TO alice;
GRANT analyst TO bob;
GRANT data_engineer TO charlie;
```

## GRANT OPTION

The GRANT OPTION allows a user to re-grant their own privileges to others:

```sql
GRANT SELECT ON analytics.events TO alice WITH GRANT OPTION;
```

Use this sparingly - any user with GRANT OPTION can delegate access up to their own permission level.

## REVOKE Syntax

```sql
REVOKE privilege ON {database.table | database.* | *.*} FROM {user | role};
```

### Revoke a Specific Privilege

```sql
REVOKE INSERT ON analytics.events FROM bob;
```

### Revoke a Role from a User

```sql
REVOKE analyst FROM alice;
```

### Revoke All Privileges

```sql
REVOKE ALL ON analytics.* FROM alice;
```

## Viewing Current Grants

```sql
-- Show grants for a specific user
SHOW GRANTS FOR alice;

-- Show all grants in the system
SELECT *
FROM system.grants
WHERE user_name = 'alice';

-- Show role assignments
SELECT *
FROM system.role_grants
WHERE user_name = 'alice';
```

## Practical Example - Read-Only Reporting User

```sql
-- Create a read-only role for BI tool access
CREATE ROLE IF NOT EXISTS bi_reader;

GRANT SELECT ON analytics.* TO bi_reader;
GRANT SELECT ON dim.*       TO bi_reader;
GRANT dictGet ON dim.*      TO bi_reader;

-- Create a user and assign the role
CREATE USER IF NOT EXISTS grafana_user
    IDENTIFIED WITH sha256_password BY 'strong_password_here'
    DEFAULT ROLE bi_reader;

GRANT bi_reader TO grafana_user;
```

```sql
-- Verify what grafana_user can do
SHOW GRANTS FOR grafana_user;
```

## Summary

GRANT and REVOKE in ClickHouse give you fine-grained SQL-driven control over what users and roles can do at the database, table, and column level. Using roles as permission bundles keeps access control manageable as teams grow, and column-level privileges enable compliance use cases where certain fields must remain hidden from specific consumers.
