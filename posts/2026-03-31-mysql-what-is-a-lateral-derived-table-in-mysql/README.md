# What Is a Lateral Derived Table in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Lateral Join, Derived Table, Advanced Sql

Description: A lateral derived table in MySQL is a subquery in the FROM clause that can reference columns from preceding tables in the same FROM clause.

---

## Overview

A lateral derived table (also called a LATERAL join) is a subquery placed in the `FROM` clause that is allowed to reference columns from tables listed earlier in the same `FROM` clause. Without `LATERAL`, a derived table cannot reference columns from other tables - it is evaluated in isolation.

`LATERAL` support was added in MySQL 8.0.14.

## Basic Syntax

```sql
SELECT ...
FROM table1, LATERAL (SELECT ... FROM ... WHERE col = table1.col) AS derived_alias;

-- Or using JOIN syntax
SELECT ...
FROM table1
JOIN LATERAL (SELECT ... FROM ... WHERE col = table1.col) AS derived_alias ON TRUE;
```

## Problem Without LATERAL

Suppose you want the top 3 orders per customer:

```sql
-- This does NOT work - subquery cannot reference c.id
SELECT c.id, recent_orders.*
FROM customers c,
     (SELECT * FROM orders WHERE customer_id = c.id ORDER BY amount DESC LIMIT 3) AS recent_orders;
-- ERROR: Unknown column 'c.id' in 'where clause'
```

## Solution With LATERAL

```sql
SELECT c.id, c.name, recent_orders.*
FROM customers c
JOIN LATERAL (
  SELECT id, amount, created_at
  FROM orders
  WHERE customer_id = c.id
  ORDER BY amount DESC
  LIMIT 3
) AS recent_orders ON TRUE;
```

## Top-N Per Group Pattern

This is the most common use case for LATERAL joins:

```sql
-- Top 2 products by sales per category
SELECT cat.name AS category, top_products.product_name, top_products.total_sales
FROM categories cat
JOIN LATERAL (
  SELECT p.name AS product_name, SUM(oi.quantity * oi.price) AS total_sales
  FROM products p
  JOIN order_items oi ON oi.product_id = p.id
  WHERE p.category_id = cat.id
  GROUP BY p.id, p.name
  ORDER BY total_sales DESC
  LIMIT 2
) AS top_products ON TRUE
ORDER BY cat.name, top_products.total_sales DESC;
```

## Using LATERAL for Row Unpivoting

```sql
-- Unpivot quarterly revenue columns into rows
SELECT y.year, q.quarter, q.revenue
FROM yearly_data y
JOIN LATERAL (
  SELECT 'Q1' AS quarter, y.q1_revenue AS revenue
  UNION ALL SELECT 'Q2', y.q2_revenue
  UNION ALL SELECT 'Q3', y.q3_revenue
  UNION ALL SELECT 'Q4', y.q4_revenue
) AS q ON TRUE;
```

## LATERAL vs Correlated Subquery

Both can reference outer columns, but they differ in usage:

| Feature | Lateral Join | Correlated Subquery |
|---------|-------------|---------------------|
| Position | FROM clause | SELECT/WHERE clause |
| Returns | Multiple rows/columns | Single value (scalar) or EXISTS |
| Can join | Yes | No |
| Performance | Often better | Can be slow row-by-row |

## Using LEFT JOIN LATERAL

```sql
-- Include customers even if they have no orders
SELECT c.name, COALESCE(latest.amount, 0) AS latest_order_amount
FROM customers c
LEFT JOIN LATERAL (
  SELECT amount
  FROM orders
  WHERE customer_id = c.id
  ORDER BY created_at DESC
  LIMIT 1
) AS latest ON TRUE;
```

## Performance Considerations

Lateral joins are evaluated once per outer row, similar to correlated subqueries. Ensure the inner query uses indexes effectively:

```sql
-- Create supporting index
CREATE INDEX idx_orders_customer_amount ON orders (customer_id, amount DESC);
```

## Summary

Lateral derived tables in MySQL allow subqueries in the FROM clause to reference columns from outer tables, enabling powerful patterns like top-N per group and row unpivoting. Available since MySQL 8.0.14, they provide a cleaner and often more performant alternative to correlated subqueries and complex window function workarounds.
