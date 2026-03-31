# How to Use RENAME TABLE Statement in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, DDL, Table Management, Database Administration

Description: Learn how to use the RENAME TABLE statement in MySQL to rename one or more tables atomically, including cross-database moves and swap patterns.

---

## What Is RENAME TABLE in MySQL

The `RENAME TABLE` statement changes the name of one or more tables in a single atomic operation. MySQL holds a metadata lock during the rename, so no other session can access the involved tables mid-operation. This makes `RENAME TABLE` a safe way to swap tables, move tables between databases, or simply correct a naming mistake.

## Basic Syntax

```sql
RENAME TABLE old_name TO new_name;
```

You can rename multiple tables in a single statement:

```sql
RENAME TABLE
    table_a TO table_b,
    table_c TO table_d;
```

## Renaming a Single Table

```sql
-- Rename orders to purchase_orders
RENAME TABLE orders TO purchase_orders;
```

All indexes, triggers, and foreign key relationships on the table are preserved. However, stored procedures and views that reference the old name will break and must be updated manually.

## Renaming Multiple Tables Atomically

Renaming multiple tables in one statement is atomic. Either all renames succeed or none do:

```sql
RENAME TABLE
    customers TO customers_old,
    customers_new TO customers;
```

This pattern is commonly used for zero-downtime table swaps after a schema migration.

## Moving a Table to Another Database

`RENAME TABLE` can move a table between databases on the same MySQL instance:

```sql
-- Move archive_orders from mydb to archive_db
RENAME TABLE mydb.archive_orders TO archive_db.archive_orders;
```

This is equivalent to a cross-database table move without copying data.

## Table Swap Pattern for Zero-Downtime Migrations

A common production pattern: build a new table, populate it, then swap atomically:

```sql
-- Step 1: Create the new table with a different schema
CREATE TABLE users_new LIKE users;
ALTER TABLE users_new ADD COLUMN phone VARCHAR(20);

-- Step 2: Copy data
INSERT INTO users_new SELECT *, NULL FROM users;

-- Step 3: Atomic swap
RENAME TABLE
    users TO users_old,
    users_new TO users;

-- Step 4: Verify and drop old table
DROP TABLE users_old;
```

Tools like `pt-online-schema-change` and `gh-ost` automate this pattern.

## RENAME TABLE vs ALTER TABLE ... RENAME

Both are equivalent for renaming a single table:

```sql
-- These are equivalent
RENAME TABLE orders TO purchase_orders;
ALTER TABLE orders RENAME TO purchase_orders;
```

`RENAME TABLE` is preferred when swapping multiple tables because it allows multi-table atomic renames, which `ALTER TABLE` does not support.

## Permissions Required

The user must have `ALTER` and `DROP` privileges on the original table, and `CREATE` and `INSERT` privileges on the target schema:

```sql
GRANT ALTER, DROP ON mydb.orders TO 'app_user'@'%';
GRANT CREATE, INSERT ON mydb.* TO 'app_user'@'%';
```

## Checking and Updating Dependent Objects

After renaming, update objects that reference the old table name:

```sql
-- Find views referencing the old table name
SELECT table_name, view_definition
FROM information_schema.views
WHERE view_definition LIKE '%orders%'
  AND table_schema = 'mydb';

-- Recreate the view with the new table name
CREATE OR REPLACE VIEW order_summary AS
    SELECT customer_id, COUNT(*) AS total
    FROM purchase_orders
    GROUP BY customer_id;
```

## Summary

The `RENAME TABLE` statement provides an atomic way to rename or swap tables in MySQL. It supports multi-table renames in one operation, cross-database moves, and is the foundation of zero-downtime table swap patterns used in production migrations. Always update views, stored procedures, and application code that reference the old table name after a rename.
