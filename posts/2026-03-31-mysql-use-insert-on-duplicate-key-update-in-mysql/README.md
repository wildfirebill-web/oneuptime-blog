# How to Use INSERT ... ON DUPLICATE KEY UPDATE in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Insert On Duplicate Key, Upsert, Sql, Database

Description: Learn how to use INSERT ... ON DUPLICATE KEY UPDATE in MySQL to insert a new row or update an existing one when a unique key conflict occurs.

---

## Introduction

`INSERT ... ON DUPLICATE KEY UPDATE` is MySQL's built-in upsert mechanism. When an inserted row would violate a `PRIMARY KEY` or `UNIQUE` constraint, MySQL updates the existing row instead of raising an error. This is useful for syncing data, maintaining counters, and caching results without the need for a separate SELECT-then-INSERT-or-UPDATE logic.

## Basic Syntax

```sql
INSERT INTO table_name (column1, column2, ...)
VALUES (value1, value2, ...)
ON DUPLICATE KEY UPDATE
  column1 = new_value1,
  column2 = new_value2;
```

## Simple Upsert Example

Insert a product or update its price if it already exists:

```sql
INSERT INTO products (sku, name, price)
VALUES ('ABC123', 'Laptop Stand', 29.99)
ON DUPLICATE KEY UPDATE
  price = 29.99;
```

If `sku` is unique and 'ABC123' already exists, the price is updated instead of raising a duplicate error.

## Using VALUES() to Reference the Inserted Value

The `VALUES()` function references the value that was being inserted:

```sql
INSERT INTO products (sku, name, price)
VALUES ('ABC123', 'Laptop Stand', 29.99)
ON DUPLICATE KEY UPDATE
  price = VALUES(price),
  name = VALUES(name);
```

Note: `VALUES()` is deprecated in MySQL 8.0.20+. Use aliases instead (see below).

## Using Column Aliases (MySQL 8.0.20+)

```sql
INSERT INTO products (sku, name, price)
VALUES ('ABC123', 'Laptop Stand', 29.99) AS new_row
ON DUPLICATE KEY UPDATE
  price = new_row.price,
  name = new_row.name;
```

## Incrementing a Counter

A common use case is tracking view counts or event counters:

```sql
INSERT INTO page_views (url, view_count)
VALUES ('/home', 1)
ON DUPLICATE KEY UPDATE
  view_count = view_count + 1;
```

## Updating Only Specific Columns

You can choose to update only some columns on conflict:

```sql
INSERT INTO user_preferences (user_id, theme, language, updated_at)
VALUES (42, 'dark', 'en', NOW())
ON DUPLICATE KEY UPDATE
  theme = VALUES(theme),
  updated_at = NOW();
-- language is NOT updated on duplicate
```

## Bulk Upsert with Multiple Rows

```sql
INSERT INTO inventory (product_id, warehouse_id, quantity)
VALUES
  (1, 10, 100),
  (2, 10, 200),
  (3, 10, 150)
ON DUPLICATE KEY UPDATE
  quantity = VALUES(quantity);
```

## LAST_INSERT_ID() Behavior

- If a new row is inserted: `LAST_INSERT_ID()` returns the new auto-increment ID.
- If an existing row is updated: `LAST_INSERT_ID()` returns 0 (not the existing ID).
- Use `id = LAST_INSERT_ID(id)` in the UPDATE clause to make `LAST_INSERT_ID()` always return the row's ID:

```sql
INSERT INTO items (name, value)
VALUES ('key1', 100)
ON DUPLICATE KEY UPDATE
  id = LAST_INSERT_ID(id),
  value = VALUES(value);

SELECT LAST_INSERT_ID(); -- returns the ID of the affected row
```

## Affected Rows Count

- INSERT of a new row: `affected_rows = 1`
- UPDATE of an existing row: `affected_rows = 2`
- No change (update sets same values): `affected_rows = 0`

## ON DUPLICATE KEY UPDATE vs REPLACE INTO

Both handle unique constraint conflicts, but differently:

- `INSERT ... ON DUPLICATE KEY UPDATE`: updates the existing row, preserving other columns and keeping the same primary key.
- `REPLACE INTO`: deletes the old row and inserts a new one (all unspecified columns reset to defaults).

```sql
-- REPLACE deletes and re-inserts (new auto-increment ID, columns reset)
REPLACE INTO products (sku, name, price) VALUES ('ABC123', 'Stand', 29.99);

-- ON DUPLICATE KEY UPDATE modifies in place (same ID, other columns preserved)
INSERT INTO products (sku, name, price) VALUES ('ABC123', 'Stand', 29.99)
ON DUPLICATE KEY UPDATE price = VALUES(price);
```

## Performance Note

This statement works only on tables with PRIMARY KEY or UNIQUE constraints. The conflict detection is based on those constraints. Ensure the columns that could conflict are indexed.

## Summary

`INSERT ... ON DUPLICATE KEY UPDATE` provides a clean upsert pattern in MySQL. It inserts a new row if no conflict exists or updates specified columns if a unique key is violated. It is more efficient than separate SELECT-then-INSERT-or-UPDATE logic and preserves the existing row's other column values.
