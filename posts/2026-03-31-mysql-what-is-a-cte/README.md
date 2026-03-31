# What Is a CTE (Common Table Expression) in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, CTE, Common Table Expression, SQL, Query

Description: A CTE in MySQL is a named temporary result set defined with a WITH clause that can be referenced within the same query for improved readability and reuse.

---

## Overview

A Common Table Expression (CTE) is a named temporary result set that you define within a SQL statement using the `WITH` clause. It exists only for the duration of the query and can be referenced one or more times within that query. CTEs were introduced in MySQL 8.0 and serve as an alternative to subqueries and derived tables that is often more readable and easier to debug.

## Basic CTE Syntax

```sql
WITH cte_name AS (
  SELECT ...
)
SELECT * FROM cte_name;
```

## Simple Example

Instead of a nested subquery:

```sql
-- Without CTE (harder to read)
SELECT customer_id, total_spent
FROM (
  SELECT customer_id, SUM(total) AS total_spent
  FROM orders
  GROUP BY customer_id
) AS summary
WHERE total_spent > 500;

-- With CTE (cleaner)
WITH order_summary AS (
  SELECT customer_id, SUM(total) AS total_spent
  FROM orders
  GROUP BY customer_id
)
SELECT customer_id, total_spent
FROM order_summary
WHERE total_spent > 500;
```

## Multiple CTEs in One Query

You can chain multiple CTEs separated by commas:

```sql
WITH
high_value_customers AS (
  SELECT customer_id, SUM(total) AS lifetime_value
  FROM orders
  GROUP BY customer_id
  HAVING lifetime_value > 1000
),
recent_customers AS (
  SELECT DISTINCT customer_id
  FROM orders
  WHERE created_at >= NOW() - INTERVAL 30 DAY
)
SELECT
  c.id,
  c.name,
  h.lifetime_value
FROM customers c
JOIN high_value_customers h ON h.customer_id = c.id
JOIN recent_customers r ON r.customer_id = c.id;
```

Each CTE can reference any CTE defined before it in the list.

## CTEs in DML Statements

CTEs work with `UPDATE`, `DELETE`, and `INSERT ... SELECT`:

```sql
-- Delete duplicate records, keeping the lowest ID
WITH duplicates AS (
  SELECT id
  FROM (
    SELECT id,
           ROW_NUMBER() OVER (PARTITION BY email ORDER BY id) AS rn
    FROM customers
  ) t
  WHERE rn > 1
)
DELETE FROM customers WHERE id IN (SELECT id FROM duplicates);
```

## CTE vs Subquery vs View

| Feature | CTE | Derived Table | View |
|---|---|---|---|
| Named | Yes | No | Yes |
| Scope | Single statement | Single clause | Persistent in DB |
| Reusable in query | Yes (multi-reference) | No | N/A |
| Recursive | Yes | No | No |
| Stored in DB | No | No | Yes |

## Referencing a CTE Multiple Times

A major advantage of CTEs over subqueries is that you can reference the same result multiple times without re-executing the query:

```sql
WITH monthly_sales AS (
  SELECT
    DATE_FORMAT(created_at, '%Y-%m') AS month,
    SUM(total) AS revenue
  FROM orders
  GROUP BY month
)
SELECT
  m.month,
  m.revenue,
  LAG(m.revenue) OVER (ORDER BY m.month) AS prev_revenue,
  m.revenue - LAG(m.revenue) OVER (ORDER BY m.month) AS growth
FROM monthly_sales m
ORDER BY m.month;
```

## Summary

CTEs in MySQL use the `WITH` clause to define named temporary result sets that last only for the duration of a single query. They improve readability by breaking complex queries into named, logical steps and allow multiple references to the same derived dataset. CTEs also unlock recursive queries for hierarchical data. They are available from MySQL 8.0 and are the modern alternative to deeply nested subqueries.
