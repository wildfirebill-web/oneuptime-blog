# How to Use Subqueries in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, SELECT, Subquery

Description: Learn how to write scalar, IN, and FROM subqueries in ClickHouse, understand correlated subquery limits, and use CTEs as a cleaner alternative.

---

Subqueries - SELECT statements nested inside another query - let you express complex logic in a single statement. ClickHouse supports scalar subqueries, FROM subqueries (derived tables), and IN subqueries. However, correlated subqueries (where the inner query references the outer query's columns) have limited support and are often better rewritten as JOINs or Common Table Expressions. This post covers all subquery forms with working examples.

## Scalar Subqueries

A scalar subquery returns a single value and can be used anywhere a constant or expression is expected.

```sql
-- Compare each user's spend to the overall average
SELECT
    user_id,
    sum(value)                                AS user_total,
    (SELECT avg(value) FROM events WHERE event_type = 'buy') AS global_avg,
    sum(value) - (SELECT avg(value) FROM events WHERE event_type = 'buy') AS diff
FROM events
WHERE event_type = 'buy'
GROUP BY user_id
ORDER BY user_total DESC
LIMIT 20;
```

Scalar subqueries are executed once and their result is cached for the duration of the outer query.

```sql
-- Use a scalar subquery in WHERE
SELECT event_id, user_id, value
FROM events
WHERE value > (SELECT avg(value) FROM events WHERE event_type = 'buy')
  AND event_type = 'buy';
```

## FROM Subqueries (Derived Tables)

A subquery in the FROM clause acts as a temporary, unnamed table. This is one of the most flexible and widely used subquery forms.

```sql
-- Aggregate over a pre-filtered and enriched set of rows
SELECT
    event_date,
    sum(daily_revenue) AS total_revenue,
    avg(daily_revenue) AS avg_daily_revenue
FROM (
    SELECT
        toDate(created_at) AS event_date,
        sum(value)         AS daily_revenue
    FROM events
    WHERE event_type = 'buy'
    GROUP BY event_date
)
GROUP BY event_date
ORDER BY event_date;
```

```sql
-- Two-level aggregation: first aggregate by user, then by tier
SELECT
    tier,
    count()    AS user_count,
    avg(spend) AS avg_spend
FROM (
    SELECT
        user_id,
        sum(value) AS spend,
        multiIf(sum(value) < 100,  'low',
                sum(value) < 1000, 'mid',
                'high')            AS tier
    FROM events
    WHERE event_type = 'buy'
    GROUP BY user_id
)
GROUP BY tier
ORDER BY avg_spend DESC;
```

## IN Subqueries

`IN` with a subquery checks whether a value belongs to the result set of the inner query.

```sql
-- Find events by users who have made at least one purchase
SELECT event_id, user_id, event_type, created_at
FROM events
WHERE user_id IN (
    SELECT DISTINCT user_id
    FROM events
    WHERE event_type = 'buy'
)
ORDER BY created_at DESC
LIMIT 100;
```

```sql
-- Exclude bot users
SELECT user_id, count() AS event_count
FROM events
WHERE user_id NOT IN (
    SELECT user_id FROM bot_users
)
GROUP BY user_id
ORDER BY event_count DESC
LIMIT 50;
```

For large IN subquery results, ClickHouse builds a hash set in memory. If the inner query produces millions of rows, consider using a JOIN instead.

## Correlated Subqueries - Limited Support

ClickHouse has limited support for correlated subqueries (where the inner query references a column from the outer query). Simple cases may work but complex correlated subqueries are often not supported and should be rewritten.

```sql
-- This pattern may work for simple cases but is not recommended
-- Prefer a JOIN or CTE instead:

-- Avoid (potentially unsupported or slow):
-- SELECT e.user_id, e.value
-- FROM events e
-- WHERE e.value = (
--     SELECT max(value) FROM events WHERE user_id = e.user_id
-- );

-- Preferred equivalent using a JOIN
SELECT e.user_id, e.value
FROM events e
INNER JOIN (
    SELECT user_id, max(value) AS max_val
    FROM events
    GROUP BY user_id
) m ON e.user_id = m.user_id AND e.value = m.max_val;
```

## Common Table Expressions as an Alternative

CTEs (WITH clauses) make complex subquery logic much more readable and allow reuse of the same derived table multiple times.

```sql
-- CTE replaces a repeated FROM subquery
WITH buyer_totals AS (
    SELECT
        user_id,
        sum(value)  AS total_spent,
        count()     AS order_count
    FROM events
    WHERE event_type = 'buy'
    GROUP BY user_id
),
high_value_buyers AS (
    SELECT user_id, total_spent
    FROM buyer_totals
    WHERE total_spent > 500
)
SELECT
    h.user_id,
    h.total_spent,
    b.order_count
FROM high_value_buyers h
INNER JOIN buyer_totals b ON h.user_id = b.user_id
ORDER BY h.total_spent DESC;
```

## JOIN as a Correlated Subquery Alternative

Most correlated subquery patterns can be rewritten as JOINs, which ClickHouse handles efficiently.

```sql
-- Top purchase per user - using JOIN instead of correlated subquery
SELECT e.user_id, e.event_id, e.value, e.created_at
FROM events e
INNER JOIN (
    SELECT user_id, max(value) AS max_value
    FROM events
    WHERE event_type = 'buy'
    GROUP BY user_id
) top ON e.user_id = top.user_id
       AND e.value  = top.max_value
WHERE e.event_type = 'buy'
ORDER BY e.user_id;
```

## Summary

ClickHouse supports scalar subqueries, FROM (derived table) subqueries, and IN subqueries effectively. Correlated subqueries have limited support and should be rewritten as JOINs or CTEs for both compatibility and performance. Use CTEs to break complex multi-step logic into readable, reusable named queries, and prefer IN with subqueries over repeated OR conditions when filtering on a dynamic set of values.
