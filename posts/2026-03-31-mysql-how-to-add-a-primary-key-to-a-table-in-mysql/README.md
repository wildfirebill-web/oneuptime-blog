# How to Add a Primary Key to a Table in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Ddl, Constraints, Schema Design

Description: Learn how to add a primary key to an existing MySQL table using ALTER TABLE, including handling tables with duplicate values and adding composite keys.

---

## What Is a Primary Key?

A primary key uniquely identifies each row in a table. MySQL enforces two rules on primary key columns:
- All values must be unique
- No value can be NULL

Each table can have at most one primary key, but it can span multiple columns (composite key).

## Adding a Primary Key with ALTER TABLE

```sql
ALTER TABLE table_name
ADD PRIMARY KEY (column_name);
```

## Simple Single-Column Primary Key

```sql
-- If the table doesn't have a primary key yet
ALTER TABLE users
ADD PRIMARY KEY (id);
```

The `id` column must contain no NULL values and no duplicates before you can add the primary key.

## Adding a Primary Key to a New AUTO_INCREMENT Column

If the table has no suitable unique column, add one:

```sql
ALTER TABLE legacy_table
ADD COLUMN id INT NOT NULL AUTO_INCREMENT FIRST,
ADD PRIMARY KEY (id);
```

## Composite Primary Key

A composite primary key spans multiple columns:

```sql
ALTER TABLE order_items
ADD PRIMARY KEY (order_id, product_id);
```

Both columns together must be unique. Neither column individually needs to be unique.

## Checking for Duplicate Values Before Adding the Key

Before adding a primary key, verify there are no duplicates or NULLs in the candidate column:

```sql
-- Check for NULLs
SELECT COUNT(*) FROM users WHERE id IS NULL;

-- Check for duplicates
SELECT id, COUNT(*) AS cnt
FROM users
GROUP BY id
HAVING cnt > 1;
```

If duplicates exist, they must be resolved (deleted or merged) before adding the primary key.

## Removing a Primary Key

```sql
ALTER TABLE users
DROP PRIMARY KEY;
```

If the primary key column is `AUTO_INCREMENT`, you must first remove that attribute:

```sql
ALTER TABLE users
MODIFY COLUMN id INT NOT NULL;  -- remove AUTO_INCREMENT

ALTER TABLE users
DROP PRIMARY KEY;
```

## Replacing an Existing Primary Key

Drop the old one and add the new one in a single statement:

```sql
ALTER TABLE orders
DROP PRIMARY KEY,
ADD PRIMARY KEY (new_id_column);
```

## Primary Key Defined at Table Creation

For comparison, the preferred way is to define the primary key when creating the table:

```sql
CREATE TABLE products (
    id   INT         NOT NULL AUTO_INCREMENT,
    sku  VARCHAR(50) NOT NULL,
    name VARCHAR(200) NOT NULL,
    PRIMARY KEY (id),
    UNIQUE KEY uq_sku (sku)
);
```

## Checking the Current Primary Key

```sql
SHOW KEYS FROM users WHERE Key_name = 'PRIMARY';

-- Or via INFORMATION_SCHEMA
SELECT COLUMN_NAME
FROM INFORMATION_SCHEMA.KEY_COLUMN_USAGE
WHERE TABLE_SCHEMA = DATABASE()
  AND TABLE_NAME = 'users'
  AND CONSTRAINT_NAME = 'PRIMARY';
```

## Online DDL Considerations

Adding a primary key in MySQL 8.0 with InnoDB may require a table rebuild (which reorganizes the clustered index). Use:

```sql
ALTER TABLE users
ADD PRIMARY KEY (id),
ALGORITHM=INPLACE, LOCK=NONE;
```

If `INPLACE` is not supported for this operation, MySQL will return an error so you can choose to proceed with `COPY`.

## Summary

Use `ALTER TABLE ... ADD PRIMARY KEY (column)` to add a primary key to an existing MySQL table. Verify there are no NULL values or duplicates in the candidate column before adding the constraint. For tables without a suitable unique column, add a new `AUTO_INCREMENT` column alongside the primary key in one `ALTER TABLE` statement. Composite primary keys span multiple columns and are added the same way with a comma-separated column list.
