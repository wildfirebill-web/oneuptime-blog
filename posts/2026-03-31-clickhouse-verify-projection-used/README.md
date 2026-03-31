# How to Verify ClickHouse Is Using Your Projection

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Projection, EXPLAIN, Query Optimization, Debugging

Description: Learn how to verify that ClickHouse is actually using your projections at query time using EXPLAIN and system tables.

---

## Why Verification Matters

Adding a projection does not guarantee it will be used. ClickHouse's query planner selects a projection only when it determines the projection will reduce work. If your query shape does not match the projection definition, ClickHouse silently falls back to the base table scan.

## Method 1: EXPLAIN with Indexes

```sql
EXPLAIN indexes = 1
SELECT status_code, count()
FROM http_logs
WHERE status_code = 200
  AND toStartOfHour(timestamp) = toStartOfHour(now())
GROUP BY status_code;
```

In the output, look for:
- `ReadFromProjection` - the projection is being used
- `Projection: <name>` - shows which projection was selected
- `Marks: N` - lower marks means fewer granules read

If you see `ReadFromMergeTree` without a projection reference, the projection was not selected.

## Method 2: EXPLAIN PIPELINE

```sql
EXPLAIN PIPELINE
SELECT service, avg(response_time_ms)
FROM http_logs
GROUP BY service;
```

A pipeline referencing projection parts confirms the projection is active.

## Method 3: Query Log - used_projections Column

After running a query:

```sql
SELECT
    query,
    used_projections,
    read_rows,
    read_bytes
FROM system.query_log
WHERE type = 'QueryFinish'
  AND query LIKE '%FROM http_logs%'
ORDER BY event_time DESC
LIMIT 5;
```

`used_projections` lists the projection names that contributed to the query result.

## Method 4: Comparing Read Rows

Run the query twice - once against the base table (bypassing projections) and once normally:

```sql
-- Force base table scan (bypass projections)
SELECT count()
FROM http_logs
SETTINGS optimize_use_projections = 0;

-- Normal query (projections enabled)
SELECT count()
FROM http_logs;
```

If the normal query reads dramatically fewer rows, the projection is being used.

## Common Reasons Projections Are Not Selected

1. **Query columns not covered**: the projection does not include all columns referenced in SELECT, WHERE, or GROUP BY
2. **GROUP BY mismatch**: for aggregate projections, the query GROUP BY must match or extend the projection's GROUP BY
3. **WHERE filter incompatible**: filters reference columns not in the projection
4. **Projection not materialized**: the `MATERIALIZE PROJECTION` mutation hasn't completed yet

Check materialization status:

```sql
SELECT name, parts_to_do, is_done, latest_fail_reason
FROM system.mutations
WHERE command LIKE '%MATERIALIZE PROJECTION%'
  AND table = 'http_logs';
```

## Disabling Projection Selection for Testing

```sql
-- Disable to force a base table scan for benchmarking
SELECT count()
FROM http_logs
SETTINGS optimize_use_projections = 0;
```

Compare query duration and rows read with and without projections to quantify the speedup.

## Summary

Verify projection usage with `EXPLAIN indexes = 1` before deploying to production, and confirm via `system.query_log.used_projections` after execution. If a projection is not being selected, audit the query shape against the projection definition - every column in SELECT and WHERE must be present, and GROUP BY structures must align for aggregate projections.
