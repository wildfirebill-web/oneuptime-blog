# How to Use SHOW PROCEDURE STATUS in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Stored Procedure, Schema

Description: Learn how to use SHOW PROCEDURE STATUS in MySQL to list stored procedures, view metadata, filter by database, and inspect creation details.

---

## What Is SHOW PROCEDURE STATUS

`SHOW PROCEDURE STATUS` lists all stored procedures visible to the current user along with metadata such as the database they belong to, creation and modification times, the definer, character set, and security type. It is the primary way to quickly audit what stored procedures exist on a MySQL server.

```sql
SHOW PROCEDURE STATUS;
SHOW PROCEDURE STATUS LIKE 'pattern';
SHOW PROCEDURE STATUS WHERE condition;
```

## Basic Usage

```sql
SHOW PROCEDURE STATUS\G
```

```text
*************************** 1. row ***************************
                  Db: myapp_db
                Name: calculate_order_total
                Type: PROCEDURE
             Definer: app_user@localhost
           Modified: 2024-03-15 09:22:11
            Created: 2024-01-10 14:30:00
      Security_type: DEFINER
             Comment: Calculates order total including tax and discounts
character_set_client: utf8mb4
collation_connection: utf8mb4_0900_ai_ci
  Database Collation: utf8mb4_0900_ai_ci
```

## Filtering by Name Pattern

```sql
-- Find procedures with 'order' in their name
SHOW PROCEDURE STATUS LIKE '%order%';

-- Find procedures starting with 'sp_'
SHOW PROCEDURE STATUS LIKE 'sp_%';
```

## Filtering by Database

The `WHERE` clause allows filtering by database:

```sql
-- Show only procedures in myapp_db
SHOW PROCEDURE STATUS WHERE Db = 'myapp_db';

-- Show procedures created after a date
SHOW PROCEDURE STATUS WHERE Created > '2024-01-01';

-- Show procedures with INVOKER security type
SHOW PROCEDURE STATUS WHERE Security_type = 'INVOKER';
```

## Key Output Columns

- **Db**: The database the procedure belongs to
- **Name**: Procedure name
- **Type**: Always `PROCEDURE` for this command
- **Definer**: The user who created the procedure
- **Modified**: Last modification timestamp
- **Created**: Creation timestamp
- **Security_type**: Either `DEFINER` (runs with creator's privileges) or `INVOKER` (runs with caller's privileges)
- **Comment**: Optional description added when the procedure was created

## Viewing All Procedures in a Database

The most common use is listing procedures for a specific database:

```sql
SHOW PROCEDURE STATUS WHERE Db = 'myapp_db';
```

Or using `information_schema`:

```sql
SELECT ROUTINE_NAME, ROUTINE_TYPE, DEFINER, CREATED, LAST_ALTERED, SECURITY_TYPE, ROUTINE_COMMENT
FROM information_schema.ROUTINES
WHERE ROUTINE_SCHEMA = 'myapp_db'
  AND ROUTINE_TYPE = 'PROCEDURE'
ORDER BY ROUTINE_NAME;
```

## Getting the Full Procedure Definition

To see the actual stored procedure code:

```sql
SHOW CREATE PROCEDURE procedure_name;
```

```sql
SHOW CREATE PROCEDURE calculate_order_total\G
```

This returns the complete `CREATE PROCEDURE` statement including parameters and body.

## Counting Procedures Per Database

```sql
SELECT ROUTINE_SCHEMA AS db_name, COUNT(*) AS procedure_count
FROM information_schema.ROUTINES
WHERE ROUTINE_TYPE = 'PROCEDURE'
GROUP BY ROUTINE_SCHEMA
ORDER BY procedure_count DESC;
```

## Finding Procedures Using DEFINER Security

Procedures with `SECURITY DEFINER` run with the definer's privileges, which can be a security risk if the definer has elevated rights:

```sql
SHOW PROCEDURE STATUS WHERE Security_type = 'DEFINER';
```

Review these to ensure they do not expose excessive privileges to callers.

## Summary

`SHOW PROCEDURE STATUS` provides a quick overview of all stored procedures on a MySQL server, including metadata on ownership, timing, and security type. Use `WHERE Db = 'database_name'` to scope results to a specific database, and `SHOW CREATE PROCEDURE` to view the full procedure body. For cross-database audits, `information_schema.ROUTINES` offers more flexible querying.
