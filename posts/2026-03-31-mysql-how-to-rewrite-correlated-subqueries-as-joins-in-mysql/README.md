# How to Rewrite Correlated Subqueries as JOINs in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Query Optimization, Correlated Subqueries, JOIN, Performance

Description: Learn how to identify correlated subqueries in MySQL and rewrite them as JOINs to significantly improve query performance and readability.

---

## What Is a Correlated Subquery?

A correlated subquery is a subquery that references columns from the outer query. MySQL evaluates it once for each row processed by the outer query, which means its cost scales linearly with the number of outer rows - often making it very slow on large tables.

```sql
-- Correlated subquery: references o.customer_id from the outer query
SELECT customer_id, order_date, total_amount
FROM orders o
WHERE total_amount > (
  SELECT AVG(total_amount)
  FROM orders
  WHERE customer_id = o.customer_id  -- correlated reference
);
```

## How to Identify Correlated Subqueries

Use `EXPLAIN` to spot correlated subqueries. Look for `DEPENDENT SUBQUERY` in the `select_type` column:

```sql
EXPLAIN
SELECT customer_id, order_date, total_amount
FROM orders o
WHERE total_amount > (
  SELECT AVG(total_amount)
  FROM orders
  WHERE customer_id = o.customer_id
);
```

```text
+----+--------------------+-------+------+----------------+
| id | select_type        | table | type | Extra          |
+----+--------------------+-------+------+----------------+
|  1 | PRIMARY            | o     | ALL  |                |
|  2 | DEPENDENT SUBQUERY | orders| ref  | Using index    |
+----+--------------------+-------+------+----------------+
```

`DEPENDENT SUBQUERY` means the inner query re-runs for every outer row.

## Rewriting as a JOIN: Basic Pattern

The general approach is to compute the subquery result once as a derived table or CTE, then JOIN it with the outer table.

```sql
-- Original correlated subquery
SELECT customer_id, order_date, total_amount
FROM orders o
WHERE total_amount > (
  SELECT AVG(total_amount)
  FROM orders
  WHERE customer_id = o.customer_id
);

-- Rewritten as a JOIN with a derived table
SELECT o.customer_id, o.order_date, o.total_amount
FROM orders o
JOIN (
  SELECT customer_id, AVG(total_amount) AS avg_amount
  FROM orders
  GROUP BY customer_id
) AS avg_by_customer ON o.customer_id = avg_by_customer.customer_id
WHERE o.total_amount > avg_by_customer.avg_amount;
```

The derived table aggregation runs once, and the JOIN is much more efficient.

## Rewriting EXISTS Correlated Subquery

`EXISTS` correlated subqueries are common when filtering rows based on the presence of related records.

```sql
-- Correlated EXISTS: find customers who have placed at least one order
SELECT c.customer_id, c.name
FROM customers c
WHERE EXISTS (
  SELECT 1 FROM orders WHERE customer_id = c.customer_id
);

-- Rewrite as JOIN (use DISTINCT to avoid duplicates)
SELECT DISTINCT c.customer_id, c.name
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id;

-- Or use an INNER JOIN with GROUP BY
SELECT c.customer_id, c.name
FROM customers c
INNER JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.name;
```

## Rewriting NOT EXISTS as LEFT JOIN

```sql
-- Correlated NOT EXISTS: customers with no orders
SELECT c.customer_id, c.name
FROM customers c
WHERE NOT EXISTS (
  SELECT 1 FROM orders WHERE customer_id = c.customer_id
);

-- Rewrite as LEFT JOIN with NULL check
SELECT c.customer_id, c.name
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
WHERE o.customer_id IS NULL;
```

## Rewriting a Correlated Subquery in the SELECT Clause

Correlated subqueries in the SELECT list are also problematic because they run once per output row.

```sql
-- Correlated subquery in SELECT clause
SELECT
  c.customer_id,
  c.name,
  (SELECT COUNT(*) FROM orders WHERE customer_id = c.customer_id) AS order_count
FROM customers c;

-- Rewrite as LEFT JOIN with aggregation
SELECT
  c.customer_id,
  c.name,
  COALESCE(o.order_count, 0) AS order_count
FROM customers c
LEFT JOIN (
  SELECT customer_id, COUNT(*) AS order_count
  FROM orders
  GROUP BY customer_id
) AS o ON c.customer_id = o.customer_id;
```

## Comparing Execution Plans

```sql
-- Always compare EXPLAIN output before and after rewriting
EXPLAIN FORMAT=JSON
SELECT c.customer_id, c.name,
       COALESCE(o.order_count, 0) AS order_count
FROM customers c
LEFT JOIN (
  SELECT customer_id, COUNT(*) AS order_count
  FROM orders
  GROUP BY customer_id
) AS o ON c.customer_id = o.customer_id;
```

Look for reduced `rows` estimates and the absence of `DEPENDENT SUBQUERY` in the rewritten plan.

## When Rewrites Are Not Needed

Modern MySQL (8.0+) automatically converts some correlated subqueries to semi-joins or uses other optimizations. Always benchmark both versions - for small tables or simple cases, the optimizer may already handle the correlated version efficiently.

```sql
-- MySQL 8.0 optimizer hints if needed
SELECT /*+ NO_SEMIJOIN() */ ...
```

## Summary

Correlated subqueries execute once per outer row and can cause severe performance issues on large datasets. Rewriting them as JOINs with derived tables or CTEs allows MySQL to execute the inner query once and join the results efficiently. Use `EXPLAIN` to identify `DEPENDENT SUBQUERY` in your execution plans, and verify improvements with `EXPLAIN FORMAT=JSON` after rewriting to confirm the plan has improved.
