# What Is a Lateral Derived Table in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Lateral Derived Table, LATERAL, SQL, Subquery

Description: A lateral derived table in MySQL is a subquery in the FROM clause that can reference columns from tables appearing earlier in the same FROM clause.

---

## Overview

A lateral derived table (introduced in MySQL 8.0.14) is a subquery in the `FROM` clause that can reference columns from preceding tables in the same `FROM` clause. Standard derived tables (subqueries in `FROM`) are self-contained and cannot reference outer columns. Adding the `LATERAL` keyword allows the subquery to be correlated with rows from a preceding table, effectively making it a "join-aware" subquery that executes once per row of the preceding table.

## The Problem Lateral Solves

Consider fetching the top 3 orders for each customer. Without LATERAL, this requires complex workarounds:

```sql
-- Without LATERAL: requires a subquery with a correlated WHERE, which is inefficient
SELECT c.id, c.name, o.id AS order_id, o.total
FROM customers c
JOIN orders o ON o.customer_id = c.id
WHERE (
  SELECT COUNT(*)
  FROM orders o2
  WHERE o2.customer_id = c.id AND o2.total >= o.total
) <= 3;
```

## Using LATERAL for Top-N Per Group

```sql
-- With LATERAL: clean, readable, efficient
SELECT c.id, c.name, top_orders.order_id, top_orders.total
FROM customers c
JOIN LATERAL (
  SELECT id AS order_id, total
  FROM orders
  WHERE customer_id = c.id        -- references c.id from outer table
  ORDER BY total DESC
  LIMIT 3
) top_orders ON TRUE;
```

The subquery `top_orders` can reference `c.id` because of `LATERAL`. It runs once per customer row and returns the top 3 orders for that customer.

## Syntax Rules

- `LATERAL` appears before the subquery in the `FROM` clause.
- Columns from tables listed before the lateral subquery are accessible.
- Columns from tables listed after it are not (left-to-right evaluation only).
- Must be followed by an alias.
- Use `ON TRUE` or `CROSS JOIN LATERAL` when there is no additional join condition.

```sql
-- CROSS JOIN LATERAL syntax (equivalent)
SELECT c.id, top_orders.order_id
FROM customers c
CROSS JOIN LATERAL (
  SELECT id AS order_id FROM orders WHERE customer_id = c.id LIMIT 3
) top_orders;
```

## Generating Per-Row Computed Values

LATERAL is useful for computing values that depend on each row:

```sql
-- Get each product with its category's average price computed alongside
SELECT p.name, p.price, cat_avg.avg_price,
       ROUND(p.price - cat_avg.avg_price, 2) AS vs_avg
FROM products p
JOIN LATERAL (
  SELECT AVG(price) AS avg_price
  FROM products
  WHERE category_id = p.category_id
) cat_avg ON TRUE;
```

## Lateral vs Correlated Subquery in SELECT

Both can reference outer rows, but LATERAL has advantages:

```sql
-- Correlated subquery in SELECT: runs once per row but limited to scalar values
SELECT c.name,
       (SELECT SUM(total) FROM orders WHERE customer_id = c.id) AS total_spent
FROM customers c;

-- LATERAL: can return multiple columns and multiple rows
SELECT c.name, recent.total, recent.created_at
FROM customers c
JOIN LATERAL (
  SELECT total, created_at FROM orders
  WHERE customer_id = c.id
  ORDER BY created_at DESC LIMIT 1
) recent ON TRUE;
```

## Performance Considerations

Lateral derived tables execute once per row of the preceding table. Ensure the inner query uses appropriate indexes:

```sql
-- The lateral query needs an index on (customer_id, total DESC)
-- or (customer_id, created_at) depending on the ORDER BY
ALTER TABLE orders ADD INDEX idx_cust_total (customer_id, total DESC);
```

```sql
EXPLAIN SELECT c.id, top_orders.order_id
FROM customers c
JOIN LATERAL (
  SELECT id AS order_id FROM orders WHERE customer_id = c.id ORDER BY total DESC LIMIT 3
) top_orders ON TRUE;
-- Should show "index lookup" inside the lateral subquery
```

## Summary

Lateral derived tables extend standard SQL subqueries in the `FROM` clause to allow references to columns from preceding tables. They make top-N-per-group queries, per-row computations, and correlated table-valued expressions clean and efficient. Available since MySQL 8.0.14, they are the modern alternative to complex self-join workarounds and are well-supported by the MySQL optimizer when the inner query has appropriate indexes.
