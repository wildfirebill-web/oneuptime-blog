# How to Implement Real-Time Visitor Counting with ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Web Analytics, Real-Time, Visitor, HyperLogLog, Analytics

Description: Implement accurate and approximate real-time visitor counting in ClickHouse using uniqExact, uniq (HyperLogLog), and materialized views for live dashboards.

---

Real-time visitor counting requires balancing accuracy and performance. ClickHouse provides both exact and approximate counting functions. This guide shows how to count visitors in real time using the right approach for your scale.

## Schema for Visitor Events

```sql
CREATE TABLE visitor_events (
    session_id  String,
    user_id     UInt64,
    ip          IPv4,
    path        LowCardinality(String),
    country     LowCardinality(String),
    created_at  DateTime DEFAULT now()
) ENGINE = MergeTree()
PARTITION BY toDate(created_at)
ORDER BY created_at;
```

## Exact vs Approximate Counting

For small to medium datasets (millions of rows), use uniqExact:

```sql
SELECT
    uniqExact(session_id) AS exact_sessions,
    uniqExact(user_id) AS exact_users
FROM visitor_events
WHERE created_at >= now() - INTERVAL 1 HOUR;
```

For billions of rows, use uniq (HyperLogLog, approximately 1% error):

```sql
SELECT
    uniq(session_id) AS approx_sessions,
    uniq(user_id) AS approx_users
FROM visitor_events
WHERE created_at >= now() - INTERVAL 1 HOUR;
```

## Current Active Visitors (Last 5 Minutes)

```sql
SELECT uniq(session_id) AS active_visitors
FROM visitor_events
WHERE created_at >= now() - INTERVAL 5 MINUTE;
```

## Visitors per Page in Real Time

```sql
SELECT
    path,
    uniq(session_id) AS visitors,
    count() AS page_views
FROM visitor_events
WHERE created_at >= now() - INTERVAL 30 MINUTE
GROUP BY path
ORDER BY visitors DESC
LIMIT 20;
```

## Materialized View for Per-Minute Counts

Pre-aggregate per-minute visitor counts for fast time-series charts:

```sql
CREATE TABLE visitor_counts_per_minute (
    minute         DateTime,
    path           LowCardinality(String),
    sessions_state AggregateFunction(uniq, String),
    users_state    AggregateFunction(uniq, UInt64)
) ENGINE = AggregatingMergeTree()
ORDER BY (minute, path);

CREATE MATERIALIZED VIEW visitor_counts_per_minute_mv
TO visitor_counts_per_minute AS
SELECT
    toStartOfMinute(created_at) AS minute,
    path,
    uniqState(session_id) AS sessions_state,
    uniqState(user_id) AS users_state
FROM visitor_events
GROUP BY minute, path;
```

Query the materialized view:

```sql
SELECT
    minute,
    path,
    uniqMerge(sessions_state) AS visitors
FROM visitor_counts_per_minute
WHERE minute >= now() - INTERVAL 1 HOUR
GROUP BY minute, path
ORDER BY minute, visitors DESC;
```

## Visitors by Country in Real Time

```sql
SELECT
    country,
    uniq(session_id) AS visitors
FROM visitor_events
WHERE created_at >= now() - INTERVAL 1 HOUR
GROUP BY country
ORDER BY visitors DESC
LIMIT 20;
```

## Combining HyperLogLog States for Rollups

Use uniqCombined for merging states across time ranges:

```sql
SELECT
    toStartOfHour(minute) AS hour,
    uniqMerge(sessions_state) AS hourly_visitors
FROM visitor_counts_per_minute
WHERE minute >= today()
GROUP BY hour
ORDER BY hour;
```

## Summary

ClickHouse provides both exact (uniqExact) and approximate (uniq, HyperLogLog) visitor counting. For real-time dashboards, the AggregatingMergeTree + materialized view pattern pre-computes per-minute counts, giving sub-second query response times regardless of underlying data volume.
