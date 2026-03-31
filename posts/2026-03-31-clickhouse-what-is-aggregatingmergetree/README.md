# What Is AggregatingMergeTree and When to Use It

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, AggregatingMergeTree, Aggregate Function, Materialized View, Incremental Aggregation

Description: Learn what AggregatingMergeTree is in ClickHouse, how it stores partial aggregate states, and when to use it for incremental real-time aggregations.

---

`AggregatingMergeTree` is a MergeTree variant designed to store partial aggregate states rather than raw values. It is the engine of choice for materialized views that maintain real-time running aggregations over event streams.

## The Problem It Solves

Consider a table with billions of events. Computing `sum(revenue) GROUP BY hour, region` at query time scans billions of rows. With `AggregatingMergeTree`, you pre-compute and store running aggregate states that are incrementally updated as new data arrives. Queries then merge these states rather than scanning raw data.

## How AggregatingMergeTree Works

Instead of storing final aggregate values, `AggregatingMergeTree` stores binary aggregate states using `AggregateFunction` column types. When ClickHouse merges parts, it combines (merges) these states rather than re-reading the original rows.

```sql
CREATE TABLE hourly_stats (
  hour          DateTime,
  region        LowCardinality(String),
  total_revenue AggregateFunction(sum, Float64),
  unique_users  AggregateFunction(uniq, UInt32),
  p99_latency   AggregateFunction(quantile(0.99), Float64)
) ENGINE = AggregatingMergeTree()
ORDER BY (hour, region);
```

## Inserting Aggregate States

You cannot insert raw values directly. Use `-State` aggregate functions:

```sql
INSERT INTO hourly_stats
SELECT
  toStartOfHour(ts) AS hour,
  region,
  sumState(revenue)            AS total_revenue,
  uniqState(user_id)           AS unique_users,
  quantileState(0.99)(latency) AS p99_latency
FROM raw_events
GROUP BY hour, region;
```

## Querying with -Merge Functions

Use `-Merge` counterparts to read final values:

```sql
SELECT
  hour,
  region,
  sumMerge(total_revenue)     AS revenue,
  uniqMerge(unique_users)     AS dau,
  quantileMerge(0.99)(p99_latency) AS p99
FROM hourly_stats
GROUP BY hour, region
ORDER BY hour DESC;
```

## Using With Materialized Views

The typical pattern combines a materialized view (to populate the table incrementally) with `AggregatingMergeTree`:

```sql
CREATE MATERIALIZED VIEW hourly_stats_mv TO hourly_stats AS
SELECT
  toStartOfHour(ts) AS hour,
  region,
  sumState(revenue)            AS total_revenue,
  uniqState(user_id)           AS unique_users,
  quantileState(0.99)(latency) AS p99_latency
FROM raw_events
GROUP BY hour, region;
```

Every new insert into `raw_events` updates `hourly_stats` without re-scanning historical data.

## When to Use AggregatingMergeTree

Use it when:
- You need real-time aggregations over high-volume event streams
- The aggregation key set is known and bounded (hours, regions, user segments)
- You need exact or approximate distinct counts with `uniq` or `uniqExact`
- You need quantile tracking with `quantile` or `quantileTDigest`

Avoid it when:
- You need to aggregate over arbitrary ad-hoc dimensions at query time
- The aggregation must be exact and the state functions are too approximate

## Summary

`AggregatingMergeTree` stores binary partial aggregate states that ClickHouse merges during background merges and at query time. Combined with materialized views, it provides incrementally maintained real-time aggregations over event streams. Use `-State` functions to insert and `-Merge` functions to query. This pattern reduces dashboard query latency from seconds to milliseconds on tables with billions of rows.
