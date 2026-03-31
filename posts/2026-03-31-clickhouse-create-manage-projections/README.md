# How to Create and Manage Projections in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Projection, MergeTree, Query Optimization, Performance

Description: Learn how to create, inspect, and manage projections in ClickHouse to accelerate queries with alternative sort orders and pre-aggregations.

---

## What Are Projections?

Projections are alternative representations of a MergeTree table's data stored within the same table parts. Unlike materialized views, projections are automatically updated on every insert and are transparent to the query planner - ClickHouse selects them automatically when they improve query performance.

## Creating a Normal (Reorder) Projection

A reorder projection stores data sorted by a different key, enabling range scans on non-primary columns:

```sql
ALTER TABLE http_logs
    ADD PROJECTION by_status_code (
        SELECT * ORDER BY status_code, timestamp
    );
```

After adding, materialize the projection for existing data:

```sql
ALTER TABLE http_logs MATERIALIZE PROJECTION by_status_code;
```

## Creating an Aggregate Projection

An aggregate projection pre-computes aggregations, enabling instant GROUP BY queries:

```sql
ALTER TABLE http_logs
    ADD PROJECTION hourly_stats (
        SELECT
            service,
            toStartOfHour(timestamp) AS hour,
            count()                   AS request_count,
            avg(response_time_ms)     AS avg_latency,
            sum(bytes)                AS total_bytes
        GROUP BY service, hour
    );

ALTER TABLE http_logs MATERIALIZE PROJECTION hourly_stats;
```

## Listing Projections on a Table

```sql
SELECT
    name,
    query,
    type
FROM system.projections
WHERE database = currentDatabase()
  AND table = 'http_logs';
```

## Checking Projection Materialization Status

```sql
SELECT
    table,
    name,
    parts_to_do,
    is_done
FROM system.mutations
WHERE command LIKE '%MATERIALIZE PROJECTION%'
  AND table = 'http_logs';
```

## Verifying Storage Used by Projections

```sql
SELECT
    table,
    name,
    formatReadableSize(sum(bytes_on_disk)) AS size_on_disk
FROM system.projection_parts
WHERE table = 'http_logs'
GROUP BY table, name;
```

## Modifying a Projection

ClickHouse does not support altering a projection directly. Drop and recreate it:

```sql
ALTER TABLE http_logs DROP PROJECTION by_status_code;

ALTER TABLE http_logs
    ADD PROJECTION by_status_code (
        SELECT * ORDER BY status_code, service, timestamp
    );

ALTER TABLE http_logs MATERIALIZE PROJECTION by_status_code;
```

## Creating Projections at Table Creation Time

```sql
CREATE TABLE http_logs (
    timestamp     DateTime,
    service       LowCardinality(String),
    status_code   UInt16,
    response_time_ms Float64,
    PROJECTION by_service (
        SELECT * ORDER BY service, timestamp
    ),
    PROJECTION hourly_summary (
        SELECT service, toStartOfHour(timestamp) AS hour,
               count() AS cnt, avg(response_time_ms) AS avg_rt
        GROUP BY service, hour
    )
) ENGINE = MergeTree()
ORDER BY (timestamp, service);
```

## Summary

Projections in ClickHouse provide a built-in mechanism to maintain alternative data layouts and pre-computed aggregates without managing separate tables or views. Use `ALTER TABLE ... ADD PROJECTION` and `MATERIALIZE PROJECTION` to add them to existing tables, and monitor materialization progress via `system.mutations` and storage overhead via `system.projection_parts`.
