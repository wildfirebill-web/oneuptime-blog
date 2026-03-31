# How to Use AggregatingMergeTree Engine in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, AggregatingMergeTree, Table Engine, Incremental Aggregation, SQL

Description: Learn how to use AggregatingMergeTree in ClickHouse to store intermediate aggregation states for efficient incremental aggregation with any aggregate function.

---

## Overview

`AggregatingMergeTree` stores aggregate function states (partial results) rather than final values. It merges these intermediate states during background operations using the same aggregate function logic. It powers incremental aggregation pipelines where new data continuously arrives and you want to maintain running aggregates without reprocessing history.

## The State/Merge/MergeState Pattern

`AggregatingMergeTree` columns use `AggregateFunction(func, type)` as the column type. You insert with `*State` suffix functions and read with `*Merge` suffix functions.

## Creating an AggregatingMergeTree Table

```sql
CREATE TABLE user_stats_agg
(
    date          Date,
    service       LowCardinality(String),
    unique_users  AggregateFunction(uniq, UInt64),
    total_revenue AggregateFunction(sum,  Float64),
    avg_latency   AggregateFunction(avg,  Float64)
)
ENGINE = AggregatingMergeTree()
ORDER BY (date, service);
```

## Inserting Aggregate States

Use `uniqState`, `sumState`, `avgState` etc. to insert partial states:

```sql
INSERT INTO user_stats_agg
SELECT
    toDate(event_time)       AS date,
    service,
    uniqState(user_id)       AS unique_users,
    sumState(revenue)        AS total_revenue,
    avgState(response_ms)    AS avg_latency
FROM events
GROUP BY date, service;
```

## Reading with Merge Functions

Use `uniqMerge`, `sumMerge`, `avgMerge` to combine states and return final values:

```sql
SELECT
    date,
    service,
    uniqMerge(unique_users)   AS dau,
    sumMerge(total_revenue)   AS revenue,
    avgMerge(avg_latency)     AS latency_ms
FROM user_stats_agg
GROUP BY date, service
ORDER BY date DESC
```

## Materialized View Pattern

The most common architecture pairs a raw MergeTree table with an AggregatingMergeTree via a materialized view:

```sql
-- Raw events table
CREATE TABLE raw_events
(
    event_time   DateTime,
    service      String,
    user_id      UInt64,
    revenue      Float64,
    response_ms  Float64
)
ENGINE = MergeTree()
ORDER BY (service, event_time);

-- Pre-aggregated table
CREATE TABLE daily_agg
(
    date         Date,
    service      LowCardinality(String),
    unique_users AggregateFunction(uniq, UInt64),
    total_rev    AggregateFunction(sum,  Float64)
)
ENGINE = AggregatingMergeTree()
ORDER BY (date, service);

-- Materialized view
CREATE MATERIALIZED VIEW mv_daily_agg
TO daily_agg
AS
SELECT
    toDate(event_time) AS date,
    service,
    uniqState(user_id)    AS unique_users,
    sumState(revenue)     AS total_rev
FROM raw_events
GROUP BY date, service;
```

New events inserted into `raw_events` automatically update `daily_agg`.

## Querying Across Date Ranges

Since states merge correctly:

```sql
SELECT
    service,
    uniqMerge(unique_users) AS monthly_unique_users,
    sumMerge(total_rev)     AS monthly_revenue
FROM daily_agg
WHERE date BETWEEN '2024-06-01' AND '2024-06-30'
GROUP BY service
ORDER BY monthly_revenue DESC
```

## Supporting Complex Aggregations

AggregatingMergeTree supports any aggregate function that has a State/Merge counterpart:

```sql
-- Quantiles
AggregateFunction(quantiles(0.5, 0.95, 0.99), Float64)

-- HyperLogLog for approximate distinct counts
AggregateFunction(uniqHLL12, UInt64)

-- GroupBitmap for user set operations
AggregateFunction(groupBitmap, UInt64)
```

## Insert via SELECT ... FROM ... INTO

```sql
INSERT INTO daily_agg
SELECT
    date,
    service,
    uniqState(user_id) AS unique_users,
    sumState(revenue)  AS total_rev
FROM raw_events
WHERE date = today()
GROUP BY date, service;
```

## Summary

`AggregatingMergeTree` stores intermediate aggregate states that can be incrementally merged as new data arrives. The `*State` functions insert partial states and `*Merge` functions produce final aggregated results. Combine with materialized views for real-time pre-aggregation pipelines that scale to billions of events while keeping query latency in milliseconds.
