# How to Limit Results with LIMIT in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Limit, Query, Performance, Pagination

Description: Learn how to use the LIMIT clause in MySQL to restrict query results, improve performance, and implement basic pagination.

---

## What Is the LIMIT Clause

The `LIMIT` clause restricts the number of rows returned by a `SELECT` statement. It is commonly used to fetch a sample of data, retrieve the top N results, or implement pagination.

```sql
SELECT column1, column2
FROM table_name
LIMIT count;
```

## Fetching the First N Rows

To retrieve just the first 10 products from a table:

```sql
SELECT id, name, price
FROM products
LIMIT 10;
```

Without an `ORDER BY`, the rows returned are not guaranteed to be in any particular order. Always combine `LIMIT` with `ORDER BY` for deterministic results.

## Combining LIMIT with ORDER BY

```sql
-- Get the 5 most expensive products
SELECT id, name, price
FROM products
ORDER BY price DESC
LIMIT 5;
```

```sql
-- Get the 3 most recently created users
SELECT id, username, created_at
FROM users
ORDER BY created_at DESC
LIMIT 3;
```

## Using LIMIT with DELETE and UPDATE

`LIMIT` can also cap the number of rows affected by `UPDATE` and `DELETE` statements, which is useful for batched operations:

```sql
-- Delete only 1000 old log rows at a time to avoid locking the table for too long
DELETE FROM audit_logs
WHERE created_at < '2024-01-01'
ORDER BY created_at ASC
LIMIT 1000;
```

```sql
-- Update at most 500 rows in one pass
UPDATE orders
SET status = 'archived'
WHERE status = 'completed' AND updated_at < '2024-06-01'
LIMIT 500;
```

## LIMIT for Performance Testing

When debugging or exploring a large table, use `LIMIT` to avoid scanning millions of rows:

```sql
SELECT * FROM events LIMIT 5;
```

## LIMIT 1 for Existence Checks

`LIMIT 1` stops scanning as soon as one matching row is found, making it faster than returning all matches when you only care whether any rows exist:

```sql
SELECT 1
FROM orders
WHERE user_id = 42 AND status = 'pending'
LIMIT 1;
```

## LIMIT and SQL_CALC_FOUND_ROWS (Deprecated Approach)

Older applications used `SQL_CALC_FOUND_ROWS` to get the total count alongside a limited result set. This is deprecated in MySQL 8.0. Use a separate `COUNT(*)` query instead:

```sql
-- Preferred approach
SELECT COUNT(*) FROM products WHERE category = 'Electronics';
SELECT id, name FROM products WHERE category = 'Electronics' LIMIT 20;
```

## Summary

The `LIMIT` clause is one of the most frequently used MySQL features for controlling result set size. Always pair it with `ORDER BY` to get predictable results. Use it in `DELETE` and `UPDATE` statements for safe batched modifications, and rely on it for quick performance checks when exploring large tables.
