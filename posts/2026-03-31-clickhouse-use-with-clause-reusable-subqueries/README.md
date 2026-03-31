# How to Use WITH Clause for Reusable Subqueries in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, CTE, WITH Clause, SQL, Query Optimization, Subquery

Description: Learn how to use the WITH clause (CTE) in ClickHouse to write cleaner, reusable subqueries and simplify complex analytical SQL.

---

The WITH clause in ClickHouse allows you to define named subqueries (CTEs - Common Table Expressions) that can be referenced multiple times in the main query. This improves readability and can simplify complex analytical queries.

## Basic WITH Clause Syntax

```sql
WITH
    active_users AS (
        SELECT DISTINCT user_id
        FROM events
        WHERE timestamp >= now() - INTERVAL 30 DAY
    )
SELECT
    u.user_id,
    u.email,
    count(e.event_id) AS event_count
FROM users u
JOIN active_users au ON u.user_id = au.user_id
LEFT JOIN events e ON u.user_id = e.user_id
GROUP BY u.user_id, u.email
ORDER BY event_count DESC
LIMIT 100;
```

## Multiple CTEs

Define multiple named expressions in sequence:

```sql
WITH
    revenue_by_day AS (
        SELECT
            toDate(timestamp) AS day,
            sum(amount) AS daily_revenue
        FROM orders
        WHERE status = 'completed'
        GROUP BY day
    ),
    avg_revenue AS (
        SELECT avg(daily_revenue) AS mean
        FROM revenue_by_day
    )
SELECT
    r.day,
    r.daily_revenue,
    a.mean AS average_revenue,
    round(r.daily_revenue / a.mean, 2) AS ratio_to_average
FROM revenue_by_day r
CROSS JOIN avg_revenue a
ORDER BY r.day;
```

## WITH for Scalar Values

CTEs can return scalar values:

```sql
WITH
    (SELECT max(timestamp) FROM events) AS latest_event,
    (SELECT count() FROM events WHERE timestamp >= today()) AS today_count
SELECT
    latest_event,
    today_count,
    dateDiff('second', latest_event, now()) AS seconds_since_last_event;
```

## Recursive-Style Queries Using WITH

While ClickHouse does not support recursive CTEs, you can chain CTEs for multi-step transformations:

```sql
WITH
    raw_sessions AS (
        SELECT
            user_id,
            timestamp,
            dateDiff('second',
                lagInFrame(timestamp) OVER (PARTITION BY user_id ORDER BY timestamp),
                timestamp
            ) AS gap_seconds
        FROM events
    ),
    session_boundaries AS (
        SELECT
            user_id,
            timestamp,
            if(gap_seconds > 1800 OR gap_seconds IS NULL, 1, 0) AS is_new_session
        FROM raw_sessions
    )
SELECT
    user_id,
    sum(is_new_session) AS session_count
FROM session_boundaries
GROUP BY user_id
ORDER BY session_count DESC;
```

## WITH for Conditional Aggregations

Use CTEs to compute multiple rollups without repeating WHERE clauses:

```sql
WITH
    base AS (
        SELECT
            service,
            status_code,
            duration_ms,
            timestamp
        FROM http_requests
        WHERE timestamp >= now() - INTERVAL 1 HOUR
    )
SELECT
    service,
    count() AS total,
    countIf(status_code >= 500) AS errors,
    avg(duration_ms) AS avg_latency,
    quantile(0.95)(duration_ms) AS p95_latency
FROM base
GROUP BY service
ORDER BY errors DESC;
```

## Performance Notes

```text
- ClickHouse evaluates each CTE independently (not shared execution)
- For expensive CTEs referenced multiple times, consider a temporary table
- Use WITH to simplify queries, not to replace proper schema design
```

## Summary

The WITH clause in ClickHouse enables cleaner, more readable SQL by naming intermediate result sets. It is especially valuable for multi-step analytical queries where the same subquery would otherwise be repeated. For CTEs referenced many times in a heavy query, consider materializing to a temporary table for better performance.
