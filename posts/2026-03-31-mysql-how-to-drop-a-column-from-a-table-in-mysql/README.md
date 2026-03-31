# How to Drop a Column from a Table in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, DDL, ALTER TABLE, Schema Design

Description: Learn how to safely drop columns from MySQL tables using ALTER TABLE DROP COLUMN, including handling dependencies and minimizing downtime.

---

## Basic DROP COLUMN Syntax

```sql
ALTER TABLE table_name
DROP COLUMN column_name;
```

Note: `COLUMN` keyword is optional - `DROP column_name` also works, but including it is clearer.

## Simple Column Drop

```sql
ALTER TABLE users
DROP COLUMN old_phone_number;
```

This permanently removes the column and all its data. This operation cannot be undone without a backup.

## Dropping Multiple Columns

```sql
ALTER TABLE users
DROP COLUMN middle_name,
DROP COLUMN fax_number,
DROP COLUMN legacy_id;
```

Combining drops into one statement is more efficient than separate statements.

## Checking for Dependencies Before Dropping

Before dropping a column, check if any indexes or foreign keys reference it:

```sql
-- Check indexes referencing the column
SHOW INDEX FROM users;

-- Check foreign keys
SELECT
    CONSTRAINT_NAME,
    TABLE_NAME,
    COLUMN_NAME,
    REFERENCED_TABLE_NAME,
    REFERENCED_COLUMN_NAME
FROM INFORMATION_SCHEMA.KEY_COLUMN_USAGE
WHERE TABLE_SCHEMA = DATABASE()
  AND TABLE_NAME = 'users'
  AND COLUMN_NAME = 'column_to_drop';
```

## Dropping a Column with an Index

If the column has an index, you must either drop the index first or drop both together:

```sql
-- Option 1: drop index first
ALTER TABLE users
DROP INDEX idx_old_phone;

ALTER TABLE users
DROP COLUMN old_phone;

-- Option 2: drop both in one statement
ALTER TABLE users
DROP INDEX idx_old_phone,
DROP COLUMN old_phone;
```

## Dropping a Column Referenced by a Foreign Key

```sql
-- First, drop the foreign key constraint
ALTER TABLE orders
DROP FOREIGN KEY fk_orders_legacy_user;

-- Then, drop the column
ALTER TABLE orders
DROP COLUMN legacy_user_id;
```

## Dropping a Primary Key Column

You cannot drop a primary key column directly without first removing the primary key:

```sql
-- Remove the primary key constraint first
ALTER TABLE old_table
DROP PRIMARY KEY;

-- Then drop the column
ALTER TABLE old_table
DROP COLUMN old_id;
```

If `AUTO_INCREMENT` is set on the column, you must also remove that attribute:

```sql
ALTER TABLE old_table
MODIFY old_id INT;  -- remove AUTO_INCREMENT

ALTER TABLE old_table
DROP PRIMARY KEY,
DROP COLUMN old_id;
```

## Online DDL for Column Drop

In MySQL 8.0 with InnoDB, dropping a column can often be done online:

```sql
ALTER TABLE users
DROP COLUMN legacy_field,
ALGORITHM=INPLACE, LOCK=NONE;
```

If the operation does not support `INPLACE`, MySQL returns an error and does not proceed.

## Verify the Column Was Dropped

```sql
DESCRIBE users;

-- Or check information schema
SELECT COLUMN_NAME
FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_SCHEMA = DATABASE()
  AND TABLE_NAME = 'users';
```

## Best Practices

```text
1. Always take a backup or snapshot before dropping columns in production.
2. First deploy application code that no longer reads/writes the column.
3. Then drop the column in a separate deployment.
4. Check for indexes and foreign keys referencing the column beforehand.
5. Use ALGORITHM=INPLACE, LOCK=NONE to minimize downtime on large tables.
```

## Summary

`ALTER TABLE ... DROP COLUMN` permanently removes a column and its data from a MySQL table. Before dropping, verify no indexes or foreign keys reference the column, as those must be removed first. On large production tables, use `ALGORITHM=INPLACE, LOCK=NONE` with InnoDB to allow concurrent reads and writes during the operation. Always ensure application code stops referencing the column before dropping it from the schema.
