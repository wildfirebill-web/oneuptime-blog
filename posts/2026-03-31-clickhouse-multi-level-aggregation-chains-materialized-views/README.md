# How to Build Multi-Level Aggregation Chains with Materialized Views in

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Materialized View, Aggregation Chain, Rollup, Pipeline

Description: Chain multiple ClickHouse materialized views to build minute-to-hour-to-day aggregation pipelines that support fast queries at any time granularity.

---

## Why Chain Materialized Views

A single materialized view aggregates raw events into one granularity. For dashboards that need per-minute, per-hour, and per-day metrics, you have two options: scan raw data every time (slow), or chain views where hourly aggregates roll up from minute aggregates, which roll up from raw events. Chaining is far more efficient.

## Layer 1 - Raw Events

```sql
CREATE TABLE raw_events
(
    event_time DateTime,
    service LowCardinality(String),
    region LowCardinality(String),
    request_count UInt32,
    error_count UInt32,
    total_duration_ms UInt64
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(event_time)
ORDER BY (service, region, event_time);
```

## Layer 2 - Per-Minute Aggregation

```sql
CREATE TABLE agg_per_minute
(
    minute DateTime,
    service LowCardinality(String),
    region LowCardinality(String),
    request_count UInt64,
    error_count UInt64,
    total_duration_ms UInt64
)
ENGINE = SummingMergeTree
PARTITION BY toDate(minute)
ORDER BY (minute, service, region);

CREATE MATERIALIZED VIEW agg_per_minute_mv
TO agg_per_minute
AS
SELECT
    toStartOfMinute(event_time) AS minute,
    service,
    region,
    sum(request_count) AS request_count,
    sum(error_count) AS error_count,
    sum(total_duration_ms) AS total_duration_ms
FROM raw_events
GROUP BY minute, service, region;
```

## Layer 3 - Per-Hour Aggregation (reads from per-minute)

```sql
CREATE TABLE agg_per_hour
(
    hour DateTime,
    service LowCardinality(String),
    region LowCardinality(String),
    request_count UInt64,
    error_count UInt64,
    total_duration_ms UInt64
)
ENGINE = SummingMergeTree
PARTITION BY toYYYYMM(hour)
ORDER BY (hour, service, region);

-- This MV reads from agg_per_minute, not raw_events
CREATE MATERIALIZED VIEW agg_per_hour_mv
TO agg_per_hour
AS
SELECT
    toStartOfHour(minute) AS hour,
    service,
    region,
    sum(request_count) AS request_count,
    sum(error_count) AS error_count,
    sum(total_duration_ms) AS total_duration_ms
FROM agg_per_minute
GROUP BY hour, service, region;
```

## Layer 4 - Per-Day Aggregation (reads from per-hour)

```sql
CREATE TABLE agg_per_day
(
    day Date,
    service LowCardinality(String),
    region LowCardinality(String),
    request_count UInt64,
    error_count UInt64,
    total_duration_ms UInt64
)
ENGINE = SummingMergeTree
PARTITION BY toYYYYMM(day)
ORDER BY (day, service, region);

CREATE MATERIALIZED VIEW agg_per_day_mv
TO agg_per_day
AS
SELECT
    toDate(hour) AS day,
    service,
    region,
    sum(request_count) AS request_count,
    sum(error_count) AS error_count,
    sum(total_duration_ms) AS total_duration_ms
FROM agg_per_hour
GROUP BY day, service, region;
```

## Querying at Each Granularity

```sql
-- Last hour at per-minute resolution
SELECT minute, service, sum(request_count) AS rpm
FROM agg_per_minute
WHERE minute >= now() - INTERVAL 1 HOUR
GROUP BY minute, service ORDER BY minute;

-- Last week at per-hour resolution
SELECT hour, service, sum(request_count) AS rph
FROM agg_per_hour
WHERE hour >= now() - INTERVAL 7 DAY
GROUP BY hour, service ORDER BY hour;

-- Last year at per-day resolution
SELECT day, service, sum(request_count) AS rpd
FROM agg_per_day
WHERE day >= today() - 365
GROUP BY day, service ORDER BY day;
```

## Important Caveat: Chain Timing

Materialized views fire on INSERT. When inserting into `raw_events`, `agg_per_minute_mv` fires immediately. But `agg_per_hour_mv` reads from `agg_per_minute` - it only fires when data is inserted into `agg_per_minute`, which happens when `agg_per_minute_mv` triggers. This chaining works automatically.

## Summary

Multi-level aggregation chains in ClickHouse connect materialized views so that each level reads from the previous level's target table. Raw events roll into per-minute buckets, which roll into per-hour buckets, which roll into per-day buckets - all triggered automatically on insert. Queries hit the appropriate granularity table, keeping all time-range dashboard queries fast regardless of data volume.
