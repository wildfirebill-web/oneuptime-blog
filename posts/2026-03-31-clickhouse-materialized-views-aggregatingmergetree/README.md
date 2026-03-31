# How to Use Materialized Views with AggregatingMergeTree in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Materialized View, AggregatingMergeTree, Aggregation, Performance

Description: Learn how to combine materialized views with AggregatingMergeTree in ClickHouse to maintain accurate incremental aggregations over large datasets.

---

## Why AggregatingMergeTree

Regular SummingMergeTree works for simple sums, but for complex aggregates like `uniq`, `quantile`, or `avg` you need `AggregatingMergeTree`. It stores intermediate aggregation states using special `AggregateFunction` column types.

## Source Table

```sql
CREATE TABLE page_views (
    event_time DateTime,
    page_id String,
    user_id UInt64,
    session_id String,
    duration_seconds UInt32,
    country LowCardinality(String)
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_time)
ORDER BY (page_id, event_time);
```

## AggregatingMergeTree Target Table

```sql
CREATE TABLE page_views_daily_agg (
    day Date,
    page_id String,
    country LowCardinality(String),
    view_count AggregateFunction(count),
    unique_users AggregateFunction(uniq, UInt64),
    unique_sessions AggregateFunction(uniq, String),
    avg_duration AggregateFunction(avg, UInt32),
    p95_duration AggregateFunction(quantile(0.95), UInt32)
) ENGINE = AggregatingMergeTree()
PARTITION BY toYYYYMM(day)
ORDER BY (day, page_id, country);
```

## Materialized View

```sql
CREATE MATERIALIZED VIEW mv_page_views_daily
TO page_views_daily_agg
AS SELECT
    toDate(event_time) AS day,
    page_id,
    country,
    countState() AS view_count,
    uniqState(user_id) AS unique_users,
    uniqState(session_id) AS unique_sessions,
    avgState(duration_seconds) AS avg_duration,
    quantileState(0.95)(duration_seconds) AS p95_duration
FROM page_views
GROUP BY day, page_id, country;
```

## Querying with Merge Functions

Always use the corresponding `-Merge` function when reading from an `AggregatingMergeTree`:

```sql
SELECT
    day,
    page_id,
    country,
    countMerge(view_count) AS views,
    uniqMerge(unique_users) AS unique_users,
    uniqMerge(unique_sessions) AS unique_sessions,
    avgMerge(avg_duration) AS avg_duration_sec,
    quantileMerge(0.95)(p95_duration) AS p95_duration_sec
FROM page_views_daily_agg
WHERE day >= today() - 30
GROUP BY day, page_id, country
ORDER BY day DESC, views DESC;
```

## Backfilling Existing Data

Populate the aggregation table from existing raw data:

```sql
INSERT INTO page_views_daily_agg
SELECT
    toDate(event_time) AS day,
    page_id,
    country,
    countState() AS view_count,
    uniqState(user_id) AS unique_users,
    uniqState(session_id) AS unique_sessions,
    avgState(duration_seconds) AS avg_duration,
    quantileState(0.95)(duration_seconds) AS p95_duration
FROM page_views
GROUP BY day, page_id, country;
```

## Common Pitfalls

- Never use `count()` instead of `countMerge()` when querying - the raw state bytes are not meaningful as numbers.
- AggregateFunction columns can only be populated using `*State` functions and read using `*Merge` functions.
- The ORDER BY key in AggregatingMergeTree determines how partial states are merged during background merges.

## Summary

Combining `AggregatingMergeTree` with materialized views is the recommended pattern for incremental analytics in ClickHouse. It supports complex aggregates like unique counts and percentiles that simple sum trees cannot handle, while maintaining accuracy as new data arrives.
