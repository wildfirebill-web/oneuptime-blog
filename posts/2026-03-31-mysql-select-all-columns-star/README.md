# How to Select All Columns with SELECT * in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, SELECT, Query

Description: Learn when to use SELECT * in MySQL to retrieve all columns, its trade-offs, impact on performance, and when to explicitly name columns instead.

---

`SELECT *` is the quickest way to retrieve all columns from a table. While convenient during exploration, it has important trade-offs in production queries. This guide covers how it works, when to use it, and when to avoid it.

## Basic SELECT *

```sql
-- Retrieve all columns from a single table
SELECT * FROM employees;

-- With a WHERE clause
SELECT * FROM employees WHERE department = 'Engineering';

-- With ORDER BY
SELECT * FROM products WHERE stock_qty > 0 ORDER BY price ASC;
```

## SELECT * with JOINs

When joining tables, `SELECT *` returns all columns from all joined tables:

```sql
-- Returns all columns from both orders and customers
SELECT *
FROM orders o
INNER JOIN customers c ON o.customer_id = c.customer_id
WHERE o.status = 'pending';
```

If both tables have a column named `id`, the result includes both, but they may be indistinguishable by name in some clients. Use explicit aliases when joining:

```sql
-- Better: be explicit about which columns you need
SELECT
  o.order_id,
  o.total,
  o.status,
  c.name AS customer_name,
  c.email
FROM orders o
INNER JOIN customers c ON o.customer_id = c.customer_id;
```

## SELECT * from a Specific Table in a Join

Use the table-qualified form to get all columns from one table in a join:

```sql
-- All order columns, plus customer name only
SELECT o.*, c.name AS customer_name
FROM orders AS o
INNER JOIN customers AS c ON o.customer_id = c.customer_id;
```

## When SELECT * Is Acceptable

```sql
-- Ad-hoc exploration in a MySQL client or during development
SELECT * FROM users LIMIT 10;

-- Checking the structure and sample data of a new table
SELECT * FROM new_import_table LIMIT 5;

-- Small tables used internally (e.g., config, lookup tables)
SELECT * FROM feature_flags;
```

## When to Avoid SELECT *

1. Production application queries - explicit column lists prevent breaking changes when the schema changes
2. Tables with large columns - selecting TEXT, BLOB, or JSON columns you do not need wastes network bandwidth and memory
3. Views and prepared statements - schema changes to underlying tables can silently return different columns

```sql
-- Preferred in application code
SELECT
  user_id,
  username,
  email,
  created_at
FROM users
WHERE status = 'active';
```

## Performance Impact

```sql
-- Check what columns exist before using SELECT *
DESCRIBE employees;
SHOW COLUMNS FROM employees;
```

If a table has 30 columns including several TEXT or JSON columns, `SELECT *` transfers all that data even if your application only uses 5 columns. Fetching only needed columns reduces I/O, network transfer, and memory usage.

## SELECT * in Subqueries

In `EXISTS` subqueries, `SELECT *` (or `SELECT 1`) is fine and the specific columns don't matter:

```sql
-- SELECT * or SELECT 1 in EXISTS - both are equivalent
SELECT customer_id, name
FROM customers c
WHERE EXISTS (
  SELECT * FROM orders o WHERE o.customer_id = c.customer_id
);
```

## Summary

`SELECT *` is a convenient shorthand that retrieves all columns from a table. It is useful for interactive exploration and development. In production application code, prefer explicit column lists to avoid unnecessary data transfer, prevent subtle bugs when schema changes, and make the query intent clear. Use `table.*` in joins when you need all columns from one specific table while selecting individual columns from another.
