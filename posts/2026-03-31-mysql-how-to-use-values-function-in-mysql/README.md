# How to Use VALUES() Function in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, VALUES Function, ON DUPLICATE KEY, Upsert, SQL Functions

Description: Learn how to use the VALUES() function in MySQL within ON DUPLICATE KEY UPDATE clauses to reference the values being inserted.

---

## What is VALUES()

In MySQL, the `VALUES()` function is used inside an `ON DUPLICATE KEY UPDATE` clause to reference the value that was provided in the `INSERT` statement for a specific column. It allows you to update a column with the value that would have been inserted.

Syntax:

```sql
INSERT INTO table_name (col1, col2, ...)
VALUES (val1, val2, ...)
ON DUPLICATE KEY UPDATE
  col1 = VALUES(col1),
  col2 = VALUES(col2);
```

Note: `VALUES()` in this context is different from the `VALUES` keyword used to provide data rows in `INSERT` statements.

## Deprecation Notice

`VALUES()` in `ON DUPLICATE KEY UPDATE` is deprecated as of MySQL 8.0.20. The recommended replacement uses column aliases from the `AS` clause.

## Basic Usage - VALUES() (MySQL < 8.0.20)

```sql
CREATE TABLE inventory (
  product_id INT NOT NULL,
  warehouse VARCHAR(50) NOT NULL,
  stock INT NOT NULL DEFAULT 0,
  last_updated DATETIME DEFAULT NOW(),
  PRIMARY KEY (product_id, warehouse)
);

-- Insert or update stock count
INSERT INTO inventory (product_id, warehouse, stock)
VALUES (42, 'East', 150)
ON DUPLICATE KEY UPDATE
  stock = VALUES(stock),          -- Use the inserted value
  last_updated = NOW();
```

## Modern Syntax - Row Alias (MySQL 8.0.20+)

The modern approach uses `AS` to name the new row:

```sql
INSERT INTO inventory (product_id, warehouse, stock)
VALUES (42, 'East', 150) AS new_row
ON DUPLICATE KEY UPDATE
  stock = new_row.stock,
  last_updated = NOW();
```

## Bulk Upsert Example

```sql
-- Update multiple inventory records
INSERT INTO inventory (product_id, warehouse, stock)
VALUES
  (1, 'East', 100),
  (2, 'East', 200),
  (3, 'West', 50)
AS new_data
ON DUPLICATE KEY UPDATE
  stock = new_data.stock,
  last_updated = NOW();
```

## Using VALUES() for Increment on Conflict

```sql
-- Add to existing stock rather than replace
INSERT INTO inventory (product_id, warehouse, stock)
VALUES (42, 'East', 25)
ON DUPLICATE KEY UPDATE
  stock = stock + VALUES(stock);  -- Add incoming to existing

-- Modern equivalent
INSERT INTO inventory (product_id, warehouse, stock)
VALUES (42, 'East', 25) AS incoming
ON DUPLICATE KEY UPDATE
  stock = stock + incoming.stock;
```

## INSERT ... ON DUPLICATE KEY UPDATE vs REPLACE

```sql
-- ON DUPLICATE KEY UPDATE: updates only specified columns
INSERT INTO users (id, name, email)
VALUES (1, 'Alice', 'alice@new.com')
ON DUPLICATE KEY UPDATE email = VALUES(email);
-- Only email is updated, other columns unchanged

-- REPLACE: deletes the old row and inserts new one
REPLACE INTO users (id, name, email)
VALUES (1, 'Alice', 'alice@new.com');
-- Entire row is replaced, auto-increment triggers, FKs may break
```

## Checking Affected Rows

```sql
INSERT INTO inventory (product_id, warehouse, stock)
VALUES (42, 'East', 150)
ON DUPLICATE KEY UPDATE stock = VALUES(stock);

SELECT ROW_COUNT();
-- Returns 1 if INSERT, 2 if UPDATE, 0 if no change
```

## Practical Use Case - Tracking Page Views

```sql
CREATE TABLE page_views (
  page_path VARCHAR(255) NOT NULL,
  view_date DATE NOT NULL,
  view_count INT NOT NULL DEFAULT 0,
  PRIMARY KEY (page_path, view_date)
);

-- Increment view count, insert if new
INSERT INTO page_views (page_path, view_date, view_count)
VALUES ('/home', CURDATE(), 1)
ON DUPLICATE KEY UPDATE
  view_count = view_count + 1;
```

## Using INSERT ... SELECT with ON DUPLICATE KEY UPDATE

```sql
INSERT INTO inventory (product_id, warehouse, stock)
SELECT product_id, 'Main', available_qty FROM shipments
ON DUPLICATE KEY UPDATE
  stock = stock + VALUES(stock);
```

## Summary

`VALUES(column)` in MySQL's `ON DUPLICATE KEY UPDATE` clause references the new value being inserted for a given column, enabling true upsert behavior. This function is deprecated in MySQL 8.0.20+ and should be replaced with row alias syntax (`AS alias_name` after `VALUES(...)`) for future compatibility. Use `ON DUPLICATE KEY UPDATE` with `VALUES()` or row aliases for efficient upserts, and use `ROW_COUNT()` afterward to determine whether an insert or update occurred.
