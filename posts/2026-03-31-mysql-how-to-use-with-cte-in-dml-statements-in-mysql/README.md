# How to Use WITH (CTE) in DML Statements in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, CTE, Common Table Expressions, SQL, DML

Description: Learn how to use WITH (Common Table Expressions) in INSERT, UPDATE, DELETE, and SELECT statements in MySQL 8.0 for cleaner and more maintainable SQL.

---

## What Is a CTE in MySQL

A Common Table Expression (CTE) is a named temporary result set defined with the `WITH` clause at the beginning of a SQL statement. CTEs make complex queries more readable by breaking them into named building blocks, and in MySQL 8.0+, they can be used not only in SELECT but also in INSERT, UPDATE, and DELETE statements.

## Basic Syntax

```sql
WITH cte_name [(column_list)] AS (
    SELECT ...
)
SELECT | INSERT | UPDATE | DELETE ...;
```

Multiple CTEs can be chained with commas:

```sql
WITH
    cte1 AS (SELECT ...),
    cte2 AS (SELECT ... FROM cte1)
SELECT * FROM cte2;
```

## CTE in a SELECT Statement

A basic use case - simplify a multi-step aggregation:

```sql
WITH monthly_revenue AS (
    SELECT
        DATE_FORMAT(created_at, '%Y-%m') AS month,
        SUM(total) AS revenue
    FROM orders
    WHERE status = 'completed'
    GROUP BY month
)
SELECT
    month,
    revenue,
    revenue - LAG(revenue) OVER (ORDER BY month) AS month_over_month_change
FROM monthly_revenue
ORDER BY month;
```

## CTE in an INSERT Statement

Use a CTE to define the data source for an INSERT:

```sql
WITH new_customers AS (
    SELECT DISTINCT customer_id
    FROM orders
    WHERE created_at >= DATE_SUB(NOW(), INTERVAL 7 DAY)
      AND customer_id NOT IN (SELECT customer_id FROM loyalty_members)
)
INSERT INTO loyalty_members (customer_id, enrolled_at)
SELECT customer_id, NOW()
FROM new_customers;
```

## CTE in an UPDATE Statement

Update rows based on aggregated data from a CTE:

```sql
WITH order_totals AS (
    SELECT customer_id, SUM(total) AS lifetime_value
    FROM orders
    WHERE status = 'completed'
    GROUP BY customer_id
)
UPDATE customers c
JOIN order_totals ot ON ot.customer_id = c.id
SET c.lifetime_value = ot.lifetime_value,
    c.tier = CASE
        WHEN ot.lifetime_value >= 10000 THEN 'gold'
        WHEN ot.lifetime_value >= 1000  THEN 'silver'
        ELSE 'bronze'
    END;
```

## CTE in a DELETE Statement

Delete rows identified by a CTE:

```sql
WITH expired_sessions AS (
    SELECT id
    FROM user_sessions
    WHERE last_activity < DATE_SUB(NOW(), INTERVAL 30 DAY)
      AND remember_me = 0
)
DELETE FROM user_sessions
WHERE id IN (SELECT id FROM expired_sessions);
```

## Recursive CTE

MySQL 8.0 supports recursive CTEs for hierarchical data:

```sql
WITH RECURSIVE org_tree AS (
    -- Anchor member: top-level employees
    SELECT id, name, manager_id, 0 AS depth
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    -- Recursive member: employees reporting to the above
    SELECT e.id, e.name, e.manager_id, ot.depth + 1
    FROM employees e
    JOIN org_tree ot ON ot.id = e.manager_id
)
SELECT id, name, depth
FROM org_tree
ORDER BY depth, name;
```

## Chaining Multiple CTEs

```sql
WITH
    completed_orders AS (
        SELECT customer_id, total, created_at
        FROM orders
        WHERE status = 'completed'
    ),
    high_value_customers AS (
        SELECT customer_id, SUM(total) AS lifetime_value
        FROM completed_orders
        GROUP BY customer_id
        HAVING lifetime_value > 5000
    )
SELECT c.id, c.username, hvc.lifetime_value
FROM customers c
JOIN high_value_customers hvc ON hvc.customer_id = c.id
ORDER BY hvc.lifetime_value DESC;
```

## CTE vs Subquery vs Temporary Table

| Feature | CTE | Subquery | Temp Table |
|---|---|---|---|
| Reusable in same query | Yes | No | Yes |
| Recursive | Yes | No | No |
| Materialized | No (by default) | No | Yes |
| Readable/Named | Yes | No | Yes |
| Cross-statement use | No | No | Yes |

## Summary

CTEs defined with the `WITH` clause in MySQL 8.0 can be used in all DML statements - SELECT, INSERT, UPDATE, and DELETE. They improve query readability by naming intermediate result sets, support recursion for hierarchical data, and enable complex multi-step transformations in a single statement. Use CTEs whenever a query benefits from named intermediate results or requires recursive traversal.
