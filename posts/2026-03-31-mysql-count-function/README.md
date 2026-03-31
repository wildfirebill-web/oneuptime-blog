# How to Use the COUNT() Function in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, COUNT Function, Aggregate Function

Description: Learn how to use the MySQL COUNT() function to count rows, non-NULL values, and distinct values, with GROUP BY and conditional counting patterns.

---

`COUNT()` is MySQL's most frequently used aggregate function. It counts rows or non-NULL values and is typically used with `GROUP BY` to produce summary statistics across groups of data.

## COUNT(*) vs COUNT(column)

```sql
-- COUNT(*): counts all rows including those with NULL values
SELECT COUNT(*) AS total_rows FROM orders;

-- COUNT(column): counts only rows where the column is NOT NULL
SELECT COUNT(shipped_date) AS shipped_count FROM orders;
```

If `shipped_date` is NULL for 10 unshipped orders, `COUNT(shipped_date)` returns fewer rows than `COUNT(*)`.

## COUNT(DISTINCT column)

```sql
-- Count unique customers who placed orders
SELECT COUNT(DISTINCT customer_id) AS unique_customers
FROM orders
WHERE YEAR(order_date) = 2025;
```

## COUNT with GROUP BY

```sql
-- Count orders per customer
SELECT customer_id, COUNT(*) AS order_count
FROM orders
GROUP BY customer_id
ORDER BY order_count DESC;

-- Count employees per department
SELECT department, COUNT(*) AS headcount
FROM employees
GROUP BY department
ORDER BY headcount DESC;
```

## COUNT with HAVING

Use `HAVING` to filter groups based on the count result:

```sql
-- Find customers who placed more than 5 orders
SELECT customer_id, COUNT(*) AS order_count
FROM orders
GROUP BY customer_id
HAVING order_count > 5
ORDER BY order_count DESC;

-- Departments with fewer than 3 employees
SELECT department, COUNT(*) AS headcount
FROM employees
GROUP BY department
HAVING headcount < 3;
```

## Conditional COUNT with CASE

Count rows matching a specific condition within a group:

```sql
-- Count completed and pending orders per customer
SELECT
  customer_id,
  COUNT(*) AS total_orders,
  COUNT(CASE WHEN status = 'completed' THEN 1 END) AS completed,
  COUNT(CASE WHEN status = 'pending' THEN 1 END) AS pending
FROM orders
GROUP BY customer_id;
```

`CASE WHEN ... THEN 1 END` returns NULL for non-matching rows, which `COUNT()` skips, effectively counting only the matching rows.

## COUNT with WHERE

```sql
-- Count all active users
SELECT COUNT(*) AS active_users
FROM users
WHERE status = 'active';

-- Count orders per month in 2025
SELECT
  MONTH(order_date) AS month,
  COUNT(*) AS order_count
FROM orders
WHERE YEAR(order_date) = 2025
GROUP BY MONTH(order_date)
ORDER BY month;
```

## COUNT in Subqueries

```sql
-- Find products with more than 10 order line items
SELECT product_id, product_name
FROM products p
WHERE (
  SELECT COUNT(*) FROM order_items oi WHERE oi.product_id = p.product_id
) > 10;
```

## Performance Tips

```sql
-- COUNT(*) on InnoDB does a full scan - consider summary tables for large datasets
EXPLAIN SELECT COUNT(*) FROM orders WHERE status = 'completed'\G

-- An index on the WHERE column helps COUNT significantly
CREATE INDEX idx_orders_status ON orders (status);
```

InnoDB does not store the row count in metadata (unlike MyISAM), so `COUNT(*)` always requires a scan. With a covering index, MySQL can scan the index instead of the full table, which is faster.

## Summary

`COUNT(*)` counts all rows including NULLs; `COUNT(column)` counts only non-NULL values; `COUNT(DISTINCT column)` counts unique non-NULL values. Use `GROUP BY` to count within groups and `HAVING` to filter groups by their count. For conditional counting within groups, use `COUNT(CASE WHEN condition THEN 1 END)`. Index the columns used in `WHERE` and `GROUP BY` to avoid full table scans on large tables.
