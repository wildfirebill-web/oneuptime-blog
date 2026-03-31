# How to Use Common Table Expressions in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, SELECT, CTE, WITH, Common Table Expression

Description: Learn how to use Common Table Expressions (CTEs) in ClickHouse with WITH ... AS syntax, multiple CTEs, and recursive CTEs for readable queries.

---

Common Table Expressions (CTEs) let you name a subquery and reference it by name within the same statement. In ClickHouse, CTEs are introduced with the `WITH` clause before `SELECT`. They improve readability by breaking complex queries into named, reusable building blocks and eliminating deeply nested subqueries.

## Basic CTE Syntax

A CTE is defined with `WITH cte_name AS (subquery)` placed before the main `SELECT`. The named result can then be referenced in the `FROM` clause or joined like any table.

```sql
WITH active_users AS (
    SELECT user_id, name
    FROM users
    WHERE status = 'active'
)
SELECT
    au.user_id,
    au.name,
    count(o.order_id) AS order_count
FROM active_users AS au
LEFT JOIN orders AS o ON au.user_id = o.user_id
GROUP BY au.user_id, au.name;
```

## Multiple CTEs

You can define several CTEs in a single `WITH` clause by separating them with commas. Each subsequent CTE may reference any CTE defined before it.

```sql
WITH
    daily_revenue AS (
        SELECT
            toDate(event_time) AS day,
            sum(amount)        AS revenue
        FROM sales
        GROUP BY day
    ),
    avg_revenue AS (
        SELECT avg(revenue) AS mean_revenue
        FROM daily_revenue
    )
SELECT
    dr.day,
    dr.revenue,
    ar.mean_revenue,
    dr.revenue - ar.mean_revenue AS delta
FROM daily_revenue AS dr
CROSS JOIN avg_revenue AS ar
ORDER BY dr.day;
```

## CTE as a Scalar Value

ClickHouse also allows a CTE to represent a single scalar value using `WITH expression AS name`:

```sql
WITH
    (SELECT max(event_time) FROM events) AS latest_event
SELECT
    event_id,
    event_time,
    latest_event - event_time AS age_seconds
FROM events
WHERE event_time >= latest_event - INTERVAL 1 HOUR;
```

## Reusing a CTE Multiple Times

Unlike an inline subquery, a CTE defined once can be referenced multiple times in the same query without rewriting the subquery:

```sql
WITH recent_errors AS (
    SELECT
        service,
        count() AS error_count
    FROM logs
    WHERE level = 'error'
      AND event_time >= now() - INTERVAL 1 DAY
    GROUP BY service
)
SELECT
    r1.service,
    r1.error_count,
    r2.error_count AS prev_day_count
FROM recent_errors AS r1
LEFT JOIN (
    SELECT service, count() AS error_count
    FROM logs
    WHERE level = 'error'
      AND event_time >= now() - INTERVAL 2 DAY
      AND event_time < now() - INTERVAL 1 DAY
    GROUP BY service
) AS r2 ON r1.service = r2.service
ORDER BY r1.error_count DESC;
```

## Recursive CTEs

ClickHouse supports recursive CTEs with the `RECURSIVE` keyword. A recursive CTE references itself to iterate over hierarchical or graph-like data. The base case and the recursive step are combined with `UNION ALL`.

```sql
WITH RECURSIVE category_tree AS (
    -- Base case: top-level categories
    SELECT
        category_id,
        parent_id,
        name,
        1 AS depth
    FROM categories
    WHERE parent_id = 0

    UNION ALL

    -- Recursive step: children of already-found categories
    SELECT
        c.category_id,
        c.parent_id,
        c.name,
        ct.depth + 1 AS depth
    FROM categories AS c
    INNER JOIN category_tree AS ct ON c.parent_id = ct.category_id
)
SELECT category_id, name, depth
FROM category_tree
ORDER BY depth, category_id;
```

ClickHouse enforces a recursion depth limit (configurable via `max_recursive_cte_evaluation_depth`) to prevent infinite loops.

## CTEs vs Subqueries - When to Choose CTEs

CTEs do not automatically improve performance over equivalent subqueries - ClickHouse may inline them. The primary benefit is readability and maintainability:

```sql
-- Hard to read with nested subqueries
SELECT user_id, spend
FROM (
    SELECT user_id, sum(amount) AS spend
    FROM orders
    WHERE status = 'completed'
    GROUP BY user_id
) AS t
WHERE spend > (
    SELECT avg(total_spend)
    FROM (
        SELECT user_id, sum(amount) AS total_spend
        FROM orders
        WHERE status = 'completed'
        GROUP BY user_id
    )
);

-- Same logic, easier to read with CTEs
WITH
    user_spend AS (
        SELECT user_id, sum(amount) AS spend
        FROM orders
        WHERE status = 'completed'
        GROUP BY user_id
    ),
    avg_spend AS (
        SELECT avg(spend) AS threshold
        FROM user_spend
    )
SELECT us.user_id, us.spend
FROM user_spend AS us
CROSS JOIN avg_spend AS av
WHERE us.spend > av.threshold;
```

## Summary

ClickHouse supports CTEs via the `WITH name AS (subquery)` syntax, allowing multiple named subqueries separated by commas. Recursive CTEs are available using the `RECURSIVE` keyword with `UNION ALL` for hierarchical traversal. CTEs primarily improve query readability and structure rather than providing guaranteed performance gains, though they can simplify complex analytics by eliminating repeated subquery definitions.
