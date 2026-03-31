# How to Use CTEs with INSERT, UPDATE, and DELETE in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, CTE, DML, SQL, Query

Description: Learn how to use CTEs with INSERT, UPDATE, and DELETE statements in MySQL to write readable, multi-step data modification queries.

---

## CTEs Are Not Just for SELECT

In MySQL 8.0, CTEs can precede not only `SELECT` statements but also `INSERT`, `UPDATE`, and `DELETE` statements. This allows you to define complex filtering or transformation logic once in a named CTE and then use it in a DML operation.

## CTE with INSERT ... SELECT

Use a CTE to compute data and insert the results into a table:

```sql
WITH monthly_summary AS (
  SELECT
    customer_id,
    YEAR(order_date)  AS yr,
    MONTH(order_date) AS mo,
    COUNT(*)          AS order_count,
    SUM(total)        AS revenue
  FROM orders
  WHERE order_date >= '2024-01-01'
  GROUP BY customer_id, yr, mo
)
INSERT INTO customer_monthly_stats (customer_id, yr, mo, order_count, revenue)
SELECT customer_id, yr, mo, order_count, revenue
FROM monthly_summary;
```

The CTE handles the aggregation; the `INSERT` statement receives clean rows.

## CTE with UPDATE

MySQL does not allow referencing the target table directly in a CTE that precedes `UPDATE`, but you can join the CTE in the `UPDATE ... JOIN` syntax:

```sql
WITH avg_salary AS (
  SELECT department_id, AVG(salary) AS dept_avg
  FROM employees
  GROUP BY department_id
)
UPDATE employees e
JOIN avg_salary a ON e.department_id = a.department_id
SET e.salary_band = CASE
  WHEN e.salary > a.dept_avg * 1.2 THEN 'High'
  WHEN e.salary < a.dept_avg * 0.8 THEN 'Low'
  ELSE 'Mid'
END;
```

The CTE computes department averages once; the `UPDATE` uses them in a single join pass.

## CTE with DELETE

Use a CTE to identify rows to delete, then reference the CTE in the `DELETE ... JOIN` pattern:

```sql
WITH stale_sessions AS (
  SELECT session_id
  FROM user_sessions
  WHERE last_activity < DATE_SUB(NOW(), INTERVAL 30 DAY)
    AND is_active = 0
)
DELETE s
FROM user_sessions s
JOIN stale_sessions ss ON s.session_id = ss.session_id;
```

This removes expired, inactive sessions without embedding the logic directly in the `DELETE`.

## Chaining Multiple CTEs Before DML

You can chain multiple CTEs before a single DML statement:

```sql
WITH
  flagged_orders AS (
    SELECT order_id
    FROM orders
    WHERE status = 'pending'
      AND created_at < DATE_SUB(NOW(), INTERVAL 7 DAY)
  ),
  customer_ids AS (
    SELECT DISTINCT customer_id
    FROM orders o
    JOIN flagged_orders f ON o.order_id = f.order_id
  )
INSERT INTO notifications (customer_id, message, created_at)
SELECT
  ci.customer_id,
  'You have pending orders older than 7 days.',
  NOW()
FROM customer_ids ci;
```

## CTE for Upsert Logic

Combine a CTE with `INSERT ... ON DUPLICATE KEY UPDATE` for clean upsert logic:

```sql
WITH new_scores AS (
  SELECT player_id, SUM(points) AS total_points
  FROM game_events
  WHERE event_date = CURDATE()
  GROUP BY player_id
)
INSERT INTO leaderboard (player_id, score, last_updated)
SELECT player_id, total_points, NOW()
FROM new_scores
ON DUPLICATE KEY UPDATE
  score        = VALUES(score),
  last_updated = VALUES(last_updated);
```

## Key Limitations

- You cannot reference the table being `UPDATE`d or `DELETE`d inside the CTE definition in MySQL. Use the `JOIN` workaround instead.
- CTEs cannot be used with `REPLACE` statements in all MySQL versions; prefer `INSERT ... ON DUPLICATE KEY UPDATE`.

## Summary

CTEs in MySQL 8 work cleanly with `INSERT`, `UPDATE`, and `DELETE`. They let you separate complex filter or aggregation logic from the DML operation itself, resulting in queries that are easier to read, test, and maintain. The key pattern for `UPDATE` and `DELETE` is to join the CTE to the target table rather than embedding the CTE as a subquery in the `WHERE` clause.
