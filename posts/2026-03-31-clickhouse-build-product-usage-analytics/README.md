# How to Build Product Usage Analytics with ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Product Analytics, Usage Tracking, Feature Flag, Retention

Description: Learn how to model and query product usage events in ClickHouse to understand feature adoption, session patterns, and user retention.

---

Product usage analytics answers questions like "which features are most used?", "how long do users stay active?", and "which user segments retain best?". ClickHouse handles the high-velocity event streams that product telemetry generates and returns answers in milliseconds, enabling product managers to iterate quickly.

## Event Schema

```sql
CREATE TABLE product_events
(
    ts           DateTime64(3),
    workspace_id UInt64,
    user_id      UInt64,
    session_id   UUID,
    feature      LowCardinality(String),
    action       LowCardinality(String), -- 'open','use','complete','error'
    plan         LowCardinality(String), -- 'free','starter','pro','enterprise'
    version      LowCardinality(String)
)
ENGINE = MergeTree()
PARTITION BY toYYYYMMDD(ts)
ORDER BY (workspace_id, user_id, ts);
```

## Daily Active Users

```sql
SELECT
    toDate(ts) AS day,
    uniqExact(user_id) AS dau
FROM product_events
WHERE ts >= now() - INTERVAL 30 DAY
GROUP BY day
ORDER BY day;
```

## Feature Usage Ranking

```sql
SELECT
    feature,
    uniqExact(user_id) AS users,
    count()            AS total_events
FROM product_events
WHERE action = 'use'
  AND ts >= now() - INTERVAL 30 DAY
GROUP BY feature
ORDER BY users DESC
LIMIT 20;
```

## Session Length Distribution

```sql
SELECT
    workspace_id,
    session_id,
    dateDiff('second', min(ts), max(ts)) AS session_duration_s
FROM product_events
GROUP BY workspace_id, session_id
HAVING session_duration_s > 0
ORDER BY session_duration_s DESC
LIMIT 100;
```

## Week-1 Retention by Signup Cohort

```sql
WITH cohorts AS (
    SELECT user_id, toStartOfWeek(min(ts)) AS signup_week
    FROM product_events
    GROUP BY user_id
)
SELECT
    c.signup_week,
    uniqExact(c.user_id) AS cohort_size,
    uniqExactIf(e.user_id,
        toStartOfWeek(e.ts) = c.signup_week + INTERVAL 7 DAY
    ) AS retained_week1
FROM cohorts c
JOIN product_events e ON c.user_id = e.user_id
GROUP BY c.signup_week
ORDER BY c.signup_week;
```

## Error Rate by Feature

```sql
SELECT
    feature,
    countIf(action = 'error') * 100.0 / count() AS error_rate_pct,
    count() AS total_uses
FROM product_events
WHERE ts >= now() - INTERVAL 7 DAY
GROUP BY feature
HAVING total_uses > 100
ORDER BY error_rate_pct DESC;
```

## Plan Tier Usage Heatmap

```sql
SELECT
    plan,
    feature,
    uniqExact(user_id) AS users
FROM product_events
WHERE ts >= now() - INTERVAL 30 DAY
  AND action = 'use'
GROUP BY plan, feature
ORDER BY plan, users DESC;
```

## Summary

ClickHouse makes product usage analytics fast and affordable. Store raw product events in a single partitioned table, then query daily active users, feature adoption, session lengths, and cohort retention with straightforward SQL. Materialized views can pre-aggregate daily rollups so product dashboards stay responsive as your event volume grows into the billions.
