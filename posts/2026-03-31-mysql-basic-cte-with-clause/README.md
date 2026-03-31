# How to Write a Basic CTE with WITH Clause in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, CTE, Query, SQL, Performance

Description: Learn how to write a basic Common Table Expression using the WITH clause in MySQL to simplify complex queries and improve readability.

---

## What is a CTE?

A Common Table Expression (CTE) is a named temporary result set defined at the start of a query using the `WITH` clause. The named result can then be referenced one or more times in the main query, just like a table or view. CTEs were introduced in MySQL 8.0.

CTEs make complex queries easier to read and maintain by breaking them into logical, named building blocks. Unlike subqueries embedded in a FROM clause, a CTE is defined once at the top and can be referenced multiple times without repeating the subquery.

## Basic Syntax

```sql
WITH cte_name AS (
  SELECT ...
)
SELECT * FROM cte_name;
```

You can define multiple CTEs in one WITH clause by separating them with commas:

```sql
WITH
  cte_one AS (SELECT ...),
  cte_two AS (SELECT ... FROM cte_one ...)
SELECT * FROM cte_two;
```

## Simple Example: Named Subquery

Suppose you want to find customers whose total orders exceed $500. Without a CTE you would nest a subquery in the FROM clause:

```sql
-- Without CTE
SELECT customer_id, total
FROM (
  SELECT customer_id, SUM(amount) AS total
  FROM orders
  GROUP BY customer_id
) AS order_totals
WHERE total > 500;
```

With a CTE the intent is much clearer:

```sql
-- With CTE
WITH order_totals AS (
  SELECT customer_id, SUM(amount) AS total
  FROM orders
  GROUP BY customer_id
)
SELECT customer_id, total
FROM order_totals
WHERE total > 500;
```

Both produce identical results, but the CTE version is easier to read and debug.

## Reusing a CTE Multiple Times

One of the key benefits of a CTE is that you can reference it more than once in the same query. Imagine you want to compare each customer's total against the overall average:

```sql
WITH order_totals AS (
  SELECT customer_id, SUM(amount) AS total
  FROM orders
  GROUP BY customer_id
)
SELECT
  ot.customer_id,
  ot.total,
  avg_data.avg_total,
  ot.total - avg_data.avg_total AS diff_from_avg
FROM order_totals ot
CROSS JOIN (
  SELECT AVG(total) AS avg_total FROM order_totals
) AS avg_data;
```

The CTE `order_totals` is referenced twice: once in the main SELECT and once in the subquery that calculates the average.

## Multiple CTEs in One Query

```sql
WITH
  active_users AS (
    SELECT id, name, email
    FROM users
    WHERE status = 'active'
  ),
  recent_orders AS (
    SELECT user_id, COUNT(*) AS order_count, MAX(created_at) AS last_order
    FROM orders
    WHERE created_at >= DATE_SUB(NOW(), INTERVAL 90 DAY)
    GROUP BY user_id
  )
SELECT
  u.name,
  u.email,
  COALESCE(r.order_count, 0) AS orders_last_90_days,
  r.last_order
FROM active_users u
LEFT JOIN recent_orders r ON u.id = r.user_id
ORDER BY r.order_count DESC;
```

Each CTE is clearly named and can reference earlier CTEs in the same WITH block.

## CTEs vs Subqueries vs Views

| Feature | CTE | Subquery | View |
|---|---|---|---|
| Scope | Single query | Single query | Database-wide |
| Reusable in same query | Yes | No | Yes |
| Can be recursive | Yes | No | No |
| Stored in DB | No | No | Yes |
| Readability | High | Low | Medium |

Use a CTE when a subquery becomes complex, when you need to reuse the same derived data, or when you want your SQL to read like documentation.

## Performance Notes

In MySQL 8, non-recursive CTEs are materialised or merged depending on the optimizer. You can inspect the execution plan to see which strategy is used:

```sql
EXPLAIN FORMAT=TREE
WITH order_totals AS (
  SELECT customer_id, SUM(amount) AS total
  FROM orders
  GROUP BY customer_id
)
SELECT * FROM order_totals WHERE total > 500\G
```

If you see `<materialized_cte>` in the plan, MySQL is storing the CTE result in a temporary table. This is often fine, but if the CTE is large and only a small subset is needed, consider pushing the filter into the CTE itself:

```sql
WITH order_totals AS (
  SELECT customer_id, SUM(amount) AS total
  FROM orders
  GROUP BY customer_id
  HAVING total > 500          -- filter early
)
SELECT * FROM order_totals;
```

## Summary

A basic CTE uses the WITH clause to name a subquery and reference it one or more times in the main query. CTEs improve readability by replacing deeply nested subqueries with clearly named building blocks, and they can be chained together in a single WITH block. In MySQL 8, CTEs also serve as the foundation for recursive queries used to traverse hierarchical data.
