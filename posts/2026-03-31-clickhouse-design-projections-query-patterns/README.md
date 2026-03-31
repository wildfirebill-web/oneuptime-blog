# How to Design Projections for Common Query Patterns in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Projection, Query Optimization, MergeTree, Performance

Description: Learn how to design effective ClickHouse projections tailored to your most common query patterns for maximum query acceleration.

---

## Understand Your Query Patterns First

Before adding projections, profile your workload. Run this to find frequently-executed, slow queries:

```sql
SELECT
    normalized_query_hash,
    count() AS executions,
    avg(query_duration_ms) AS avg_ms,
    any(query) AS sample_query
FROM system.query_log
WHERE type = 'QueryFinish'
  AND query_duration_ms > 100
GROUP BY normalized_query_hash
ORDER BY executions * avg_ms DESC
LIMIT 20;
```

## Pattern 1: Filtering on a Non-Primary Key Column

Problem: your primary key is `(timestamp)` but queries often filter by `user_id`:

```sql
-- Slow without projection (full scan)
SELECT count(), avg(response_ms)
FROM events
WHERE user_id = 'u-12345';
```

Solution: add a reorder projection:

```sql
ALTER TABLE events
    ADD PROJECTION by_user (
        SELECT * ORDER BY user_id, timestamp
    );
ALTER TABLE events MATERIALIZE PROJECTION by_user;
```

## Pattern 2: Frequent GROUP BY on Low-Cardinality Columns

```sql
-- Aggregation scanned on large table
SELECT status_code, count(), avg(latency_ms)
FROM http_logs
GROUP BY status_code;
```

Add an aggregate projection:

```sql
ALTER TABLE http_logs
    ADD PROJECTION status_summary (
        SELECT status_code, count() AS cnt, avg(latency_ms) AS avg_lat
        GROUP BY status_code
    );
ALTER TABLE http_logs MATERIALIZE PROJECTION status_summary;
```

Now the query reads a tiny pre-aggregated projection rather than billions of rows.

## Pattern 3: Time-Bucketed Rollup Queries

Dashboard queries often group by hour or day:

```sql
ALTER TABLE metrics
    ADD PROJECTION hourly_rollup (
        SELECT
            metric_name,
            host,
            toStartOfHour(timestamp) AS hour,
            avg(value) AS avg_val,
            max(value) AS max_val
        GROUP BY metric_name, host, hour
    );
ALTER TABLE metrics MATERIALIZE PROJECTION hourly_rollup;
```

## Pattern 4: Multi-Column Range Scan

When queries filter on two columns not in the primary key order:

```sql
ALTER TABLE orders
    ADD PROJECTION by_region_date (
        SELECT * ORDER BY region, toDate(created_at)
    );
ALTER TABLE orders MATERIALIZE PROJECTION by_region_date;
```

## Choosing Between Normal and Aggregate Projections

Use a **normal (reorder) projection** when:
- Queries do `SELECT *` or many columns with WHERE filters
- You need per-row access in a different sort order

Use an **aggregate projection** when:
- Queries are always GROUP BY aggregations
- You can pre-compute the result and don't need raw row access

## Validating Projection Design with EXPLAIN

```sql
EXPLAIN
SELECT status_code, count()
FROM http_logs
GROUP BY status_code;
```

Look for `Projection: status_summary` in the output to confirm the projection is selected.

## Avoiding Over-Projection

Each projection increases write amplification and storage. Follow these rules:
- Add projections only for queries that run frequently (multiple times per minute) or at high data volume
- Limit to 3-5 projections per table
- Monitor `system.projection_parts` for storage growth

## Summary

Effective projection design starts with query pattern analysis. Match reorder projections to filter-heavy queries on non-key columns, and aggregate projections to frequent GROUP BY workloads. Validate with `EXPLAIN` and monitor storage to ensure projections deliver acceleration without excessive write overhead.
