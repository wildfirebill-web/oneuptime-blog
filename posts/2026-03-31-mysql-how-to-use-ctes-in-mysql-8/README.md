# How to Use CTEs in MySQL 8

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, CTE, Common Table Expressions, SQL, Query Optimization

Description: Learn how to use Common Table Expressions (CTEs) in MySQL 8 to write cleaner, more readable, and reusable SQL queries.

---

## What Are CTEs

A Common Table Expression (CTE) is a named temporary result set defined within a `WITH` clause that exists only for the duration of a single query. CTEs were introduced in MySQL 8.0 and make complex queries easier to read and maintain compared to deeply nested subqueries.

## Basic CTE Syntax

```sql
WITH cte_name AS (
  SELECT ...
)
SELECT * FROM cte_name;
```

## Simple CTE Example

```sql
WITH recent_orders AS (
  SELECT id, customer_id, total_amount, order_date
  FROM orders
  WHERE order_date >= CURDATE() - INTERVAL 30 DAY
)
SELECT customer_id, COUNT(*) AS order_count, SUM(total_amount) AS revenue
FROM recent_orders
GROUP BY customer_id
ORDER BY revenue DESC;
```

## Multiple CTEs in One Query

You can define multiple CTEs separated by commas:

```sql
WITH
active_customers AS (
  SELECT id, name, email
  FROM customers
  WHERE status = 'active'
),
monthly_orders AS (
  SELECT customer_id, SUM(total_amount) AS monthly_total
  FROM orders
  WHERE order_date >= DATE_FORMAT(NOW(), '%Y-%m-01')
  GROUP BY customer_id
)
SELECT
  c.name,
  c.email,
  COALESCE(mo.monthly_total, 0) AS this_month_spend
FROM active_customers c
LEFT JOIN monthly_orders mo ON c.id = mo.customer_id
ORDER BY this_month_spend DESC;
```

## CTE Referencing Another CTE

Later CTEs can reference earlier ones in the same `WITH` clause:

```sql
WITH
base_sales AS (
  SELECT product_id, SUM(quantity) AS total_qty, SUM(revenue) AS total_rev
  FROM sales
  WHERE year = 2025
  GROUP BY product_id
),
ranked_sales AS (
  SELECT
    product_id,
    total_qty,
    total_rev,
    RANK() OVER (ORDER BY total_rev DESC) AS revenue_rank
  FROM base_sales
)
SELECT p.name, rs.total_qty, rs.total_rev, rs.revenue_rank
FROM ranked_sales rs
JOIN products p ON rs.product_id = p.id
WHERE rs.revenue_rank <= 10;
```

## CTEs vs Subqueries

A subquery version of the same logic is harder to read:

```sql
-- Subquery version (harder to read)
SELECT customer_id, COUNT(*) AS order_count
FROM (
  SELECT id, customer_id
  FROM orders
  WHERE order_date >= CURDATE() - INTERVAL 30 DAY
) AS recent
GROUP BY customer_id;
```

The CTE version is easier to follow, especially when the same derived table is referenced multiple times.

## Using CTEs with INSERT, UPDATE, DELETE

CTEs can precede DML statements in MySQL 8:

```sql
-- Delete duplicate rows keeping the one with the highest ID
WITH duplicates AS (
  SELECT id,
         ROW_NUMBER() OVER (PARTITION BY email ORDER BY id DESC) AS rn
  FROM customers
)
DELETE FROM customers
WHERE id IN (
  SELECT id FROM duplicates WHERE rn > 1
);
```

Note: MySQL does not yet support referencing the target table of a `DELETE` directly in a CTE in all versions - use a subquery wrapper if needed.

## CTEs with Aggregation and Filtering

```sql
WITH dept_stats AS (
  SELECT
    department_id,
    AVG(salary) AS avg_salary,
    MAX(salary) AS max_salary
  FROM employees
  GROUP BY department_id
)
SELECT
  e.name,
  e.salary,
  ds.avg_salary,
  e.salary - ds.avg_salary AS salary_vs_avg
FROM employees e
JOIN dept_stats ds ON e.department_id = ds.department_id
WHERE e.salary > ds.avg_salary
ORDER BY salary_vs_avg DESC;
```

## Performance Considerations

MySQL materializes CTEs in some cases, which can affect performance. If a CTE is referenced multiple times, MySQL may compute it once and cache the result. You can use `EXPLAIN` to check:

```sql
EXPLAIN WITH recent_orders AS (
  SELECT id, customer_id FROM orders WHERE order_date > '2025-01-01'
)
SELECT * FROM recent_orders;
```

Look for "materialize" in the output to understand how MySQL handles the CTE.

## Summary

CTEs in MySQL 8 provide a clean way to decompose complex queries into readable, named steps. They support multiple definitions in a single `WITH` clause, can reference earlier CTEs, and work with SELECT, INSERT, UPDATE, and DELETE statements. For hierarchical or tree data, MySQL also supports recursive CTEs using `WITH RECURSIVE`.
