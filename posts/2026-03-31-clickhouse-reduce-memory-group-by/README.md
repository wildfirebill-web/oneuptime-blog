# How to Reduce Memory Usage of GROUP BY in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, GROUP BY, Memory Optimization, Performance, Query Tuning

Description: Reduce ClickHouse GROUP BY memory usage by tuning aggregation settings, using two-level hash tables, and spilling to disk.

---

GROUP BY queries in ClickHouse can consume large amounts of memory when aggregating over high-cardinality columns. Several settings and techniques reduce peak memory usage without sacrificing accuracy.

## Understand the Problem

ClickHouse builds a hash table in memory for GROUP BY. The size depends on the number of distinct groups and the width of aggregate states:

```sql
-- Check memory usage for a heavy GROUP BY
SELECT
  query_id,
  memory_usage,
  query
FROM system.query_log
WHERE query_like '%GROUP BY%'
  AND memory_usage > 1e9
ORDER BY memory_usage DESC
LIMIT 10;
```

## Enable Two-Level Aggregation

Two-level aggregation splits the hash table into buckets, reducing memory per bucket:

```sql
SET group_by_two_level_threshold = 100000;
SET group_by_two_level_threshold_bytes = 50000000;  -- 50MB

SELECT user_id, count()
FROM events
GROUP BY user_id;
```

ClickHouse automatically switches to two-level mode when the number of groups or memory exceeds these thresholds.

## Limit Memory and Spill to Disk

Enable spilling excess aggregation state to disk:

```sql
SET max_bytes_before_external_group_by = 2000000000;  -- 2GB
SET max_memory_usage = 4000000000;  -- 4GB hard limit

SELECT user_id, sum(amount)
FROM orders
GROUP BY user_id;
```

When the in-memory hash table exceeds `max_bytes_before_external_group_by`, ClickHouse writes partial aggregation results to disk and merges them at the end.

## Use Approximate Aggregations

For dashboards that can tolerate small errors, use approximate functions:

```sql
-- Exact: high memory
SELECT count(DISTINCT user_id) FROM events;

-- Approximate: much lower memory
SELECT uniq(user_id) FROM events;
SELECT uniqHLL12(user_id) FROM events;  -- HyperLogLog
```

## Pre-Aggregate with Materialized Views

The best way to reduce GROUP BY memory is to pre-aggregate so queries hit smaller datasets:

```sql
CREATE MATERIALIZED VIEW daily_user_counts
ENGINE = SummingMergeTree()
ORDER BY (event_date, user_id)
AS
SELECT
  toDate(event_time) AS event_date,
  user_id,
  count() AS events
FROM events
GROUP BY event_date, user_id;

-- Query the aggregated view instead
SELECT user_id, sum(events)
FROM daily_user_counts
GROUP BY user_id;
```

## Profile Memory with system.query_log

Track peak memory for GROUP BY queries:

```sql
SELECT
  formatReadableSize(memory_usage) AS memory,
  formatReadableSize(peak_memory_usage) AS peak_memory,
  query_duration_ms,
  query
FROM system.query_log
WHERE type = 'QueryFinish'
  AND query_like '%GROUP BY%'
ORDER BY peak_memory_usage DESC
LIMIT 5;
```

## Summary

GROUP BY memory usage in ClickHouse is reduced by enabling two-level aggregation with appropriate thresholds, setting `max_bytes_before_external_group_by` to allow disk spilling, using approximate aggregation functions like `uniq` instead of `count(DISTINCT)`, and pre-aggregating data with materialized views. Combining these approaches allows GROUP BY over billions of rows with predictable memory usage.
