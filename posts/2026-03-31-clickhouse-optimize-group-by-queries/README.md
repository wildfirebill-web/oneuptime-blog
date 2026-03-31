# How to Optimize GROUP BY Queries in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, GROUP BY, Query Optimization, Aggregation, Performance, Memory

Description: Learn practical techniques to speed up GROUP BY queries in ClickHouse, from key ordering to memory settings and pre-aggregation strategies.

---

GROUP BY is one of the most common operations in analytics workloads, and ClickHouse has several mechanisms to make it fast. Understanding how ClickHouse executes aggregation helps you write queries that scale to billions of rows.

## How ClickHouse Executes GROUP BY

ClickHouse uses a hash table to aggregate rows. Each worker thread builds its own partial hash table, then the results are merged. If the hash table exceeds available memory, ClickHouse can spill to disk (with `max_bytes_before_external_group_by`) or fail with an out-of-memory error.

## Order Your GROUP BY Keys by Cardinality

Put low-cardinality keys first. This reduces hash table size and improves cache locality:

```sql
-- Better: low cardinality first
SELECT status, user_id, count()
FROM requests
GROUP BY status, user_id;

-- Worse: high cardinality first
SELECT user_id, status, count()
FROM requests
GROUP BY user_id, status;
```

## Use group_by_overflow_mode

```sql
SET group_by_overflow_mode = 'any';
SET max_rows_to_group_by = 10000000;
```

With `any`, ClickHouse returns approximate results when the group count exceeds the limit instead of throwing an error, which is useful for dashboards where approximation is acceptable.

## Enable External Aggregation for Large Datasets

```sql
SET max_bytes_before_external_group_by = 4000000000; -- 4 GB
SET max_memory_usage = 8000000000;                    -- 8 GB
```

When the in-memory hash table exceeds `max_bytes_before_external_group_by`, ClickHouse spills to disk and merges results. This prevents OOM at the cost of I/O.

## Pre-aggregate with Materialized Views

For repeated GROUP BY patterns, maintain running aggregates:

```sql
CREATE TABLE requests_by_status_mv_state (
    status     String,
    total      AggregateFunction(count)
) ENGINE = AggregatingMergeTree()
ORDER BY status;

CREATE MATERIALIZED VIEW requests_by_status_mv
TO requests_by_status_mv_state
AS SELECT status, countState() AS total
FROM requests
GROUP BY status;
```

```sql
-- Query the pre-aggregated table - no scan of raw data
SELECT status, countMerge(total) AS total
FROM requests_by_status_mv_state
GROUP BY status;
```

## Use Projections for Common Aggregations

```sql
ALTER TABLE requests
ADD PROJECTION proj_by_status (
    SELECT status, count() GROUP BY status
);

ALTER TABLE requests MATERIALIZE PROJECTION proj_by_status;
```

ClickHouse will automatically use the projection when the query matches.

## Avoid GROUP BY on High-Cardinality String Columns

Aggregating on raw UUIDs or long strings is expensive. Consider pre-hashing or encoding:

```sql
SELECT cityHash64(session_id) % 1000 AS bucket, count()
FROM events
GROUP BY bucket;
```

## Summary

Optimizing GROUP BY in ClickHouse involves choosing the right key order, enabling external aggregation for large datasets, and pre-aggregating with materialized views or projections for hot query patterns. Use `EXPLAIN` and `system.query_log` to measure memory usage and execution time before and after changes.
