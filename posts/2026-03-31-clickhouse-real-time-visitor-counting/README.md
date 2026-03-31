# How to Implement Real-Time Visitor Counting with ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Real-Time Analytics, Visitor Counting, uniq, HyperLogLog, Web Analytics

Description: Learn how to implement real-time visitor counting in ClickHouse using HyperLogLog approximations, materialized views, and efficient query patterns.

---

Counting unique visitors in real time is one of the most common web analytics requirements. ClickHouse provides several approaches, from exact counts to highly efficient approximations, all capable of operating on billions of events with sub-second latency.

## The Challenge with Exact Counts

Exact unique counting requires storing all visitor IDs in memory or on disk. At scale (hundreds of millions of visitors), this becomes expensive. ClickHouse provides both exact (`uniqExact`) and approximate (`uniq`, `uniqCombined`, `uniqHLL12`) functions:

```sql
-- Exact count - accurate but memory intensive
SELECT uniqExact(visitor_id) AS exact_visitors FROM pageviews WHERE ts >= today();

-- Approximate count - 99%+ accuracy, much faster
SELECT uniq(visitor_id) AS approx_visitors FROM pageviews WHERE ts >= today();

-- HyperLogLog with configurable precision
SELECT uniqHLL12(visitor_id) AS hll_visitors FROM pageviews WHERE ts >= today();
```

For most analytics use cases, `uniq` (which uses a 65536-bucket HyperLogLog) is accurate enough and much faster.

## Real-Time Materialized View

Pre-aggregate visitor counts as data arrives to enable instant lookups:

```sql
CREATE TABLE visitor_counts_hourly (
    hour         DateTime,
    country      LowCardinality(String),
    page         String,
    visitors     AggregateFunction(uniq, String),
    pageviews    UInt64
) ENGINE = AggregatingMergeTree
ORDER BY (hour, country, page);

CREATE MATERIALIZED VIEW visitor_counts_hourly_mv
TO visitor_counts_hourly AS
SELECT
    toStartOfHour(ts)       AS hour,
    country,
    url                     AS page,
    uniqState(visitor_id)   AS visitors,
    count()                 AS pageviews
FROM pageviews
GROUP BY hour, country, page;
```

Query the pre-aggregated view instantly:

```sql
SELECT
    hour,
    country,
    uniqMerge(visitors)  AS unique_visitors,
    sum(pageviews)       AS total_pageviews
FROM visitor_counts_hourly
WHERE hour >= now() - INTERVAL 24 HOUR
GROUP BY hour, country
ORDER BY hour DESC, unique_visitors DESC;
```

## Active Visitors in the Last 5 Minutes

```sql
SELECT
    count()              AS active_events,
    uniq(visitor_id)     AS active_visitors,
    uniq(session_id)     AS active_sessions
FROM pageviews
WHERE ts >= now() - INTERVAL 5 MINUTE;
```

Expose this query via a polling endpoint (every 10-30 seconds) for a live visitor counter.

## Windowed Unique Counts

Count unique visitors across rolling windows:

```sql
SELECT
    toStartOfMinute(ts)     AS minute,
    uniq(visitor_id)        AS visitors_in_minute,
    uniqExactIf(visitor_id, ts >= now() - INTERVAL 1 HOUR) AS visitors_last_hour
FROM pageviews
WHERE ts >= now() - INTERVAL 1 HOUR
GROUP BY minute
ORDER BY minute;
```

## Tracking New vs Returning Visitors

```sql
SELECT
    toDate(first_seen)                                AS cohort_date,
    uniq(visitor_id)                                  AS new_visitors,
    uniqIf(visitor_id, first_seen < today() - 1)      AS returning_visitors
FROM (
    SELECT
        visitor_id,
        min(ts) AS first_seen
    FROM pageviews
    WHERE ts >= today() - 30
    GROUP BY visitor_id
)
GROUP BY cohort_date
ORDER BY cohort_date;
```

## Summary

ClickHouse's HyperLogLog-based `uniq` function makes real-time unique visitor counting both fast and memory-efficient. Combined with `AggregatingMergeTree` materialized views, you can serve live visitor counts with millisecond latency regardless of data volume. For truly real-time counters, combine materialized view pre-aggregation with lightweight queries against recent raw data.
