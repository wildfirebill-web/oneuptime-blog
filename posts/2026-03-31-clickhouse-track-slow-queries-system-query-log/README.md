# How to Track Slow Queries with system.query_log in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Query Optimization, system.query_log, Performance, Monitoring

Description: Learn how to use ClickHouse's system.query_log table to identify and analyze slow queries, set thresholds, and optimize performance.

---

Slow queries are one of the most common performance issues in ClickHouse. The built-in `system.query_log` table captures detailed execution statistics for every query, making it easy to find and fix bottlenecks.

## Enabling and Configuring Query Logging

By default, ClickHouse logs all queries to `system.query_log`. You can control which queries are logged and configure the log retention via `config.xml`:

```xml
<query_log>
    <database>system</database>
    <table>query_log</table>
    <flush_interval_milliseconds>7500</flush_interval_milliseconds>
    <max_size_rows>1048576</max_size_rows>
    <reserved_size_rows>8192</reserved_size_rows>
</query_log>
```

Set a minimum query duration to reduce noise - only log queries slower than a threshold:

```sql
SET log_queries_min_type = 'QUERY_FINISH';
SET log_queries_min_query_duration_ms = 1000;
```

## Finding Slow Queries

Query `system.query_log` to find the slowest queries over the last 24 hours:

```sql
SELECT
    query_duration_ms,
    read_rows,
    read_bytes,
    memory_usage,
    query
FROM system.query_log
WHERE
    type = 'QueryFinish'
    AND event_time >= now() - INTERVAL 1 DAY
    AND is_initial_query = 1
ORDER BY query_duration_ms DESC
LIMIT 20;
```

## Analyzing Slow Query Patterns

Group slow queries by normalized pattern to spot repeated offenders:

```sql
SELECT
    normalized_query_hash,
    count() AS query_count,
    avg(query_duration_ms) AS avg_duration_ms,
    max(query_duration_ms) AS max_duration_ms,
    avg(read_rows) AS avg_rows_read,
    any(query) AS sample_query
FROM system.query_log
WHERE
    type = 'QueryFinish'
    AND query_duration_ms > 5000
    AND event_time >= now() - INTERVAL 7 DAY
GROUP BY normalized_query_hash
ORDER BY avg_duration_ms DESC
LIMIT 10;
```

## Identifying Resource-Heavy Queries

Some queries may not be slow by duration but consume large amounts of memory or read many rows:

```sql
SELECT
    query,
    memory_usage,
    read_rows,
    read_bytes,
    query_duration_ms
FROM system.query_log
WHERE
    type = 'QueryFinish'
    AND event_time >= now() - INTERVAL 1 DAY
    AND (memory_usage > 1e9 OR read_bytes > 1e10)
ORDER BY memory_usage DESC
LIMIT 10;
```

## Tracking Slow Queries by User and Table

Find which users or tables are generating the most slow queries:

```sql
SELECT
    user,
    tables,
    count() AS slow_query_count,
    avg(query_duration_ms) AS avg_ms
FROM system.query_log
WHERE
    type = 'QueryFinish'
    AND query_duration_ms > 10000
    AND event_time >= now() - INTERVAL 1 DAY
GROUP BY user, tables
ORDER BY slow_query_count DESC;
```

## Setting Up Alerts with Materialized Views

Create a materialized view to track slow queries in a dedicated table for alerting:

```sql
CREATE MATERIALIZED VIEW slow_query_tracker
ENGINE = MergeTree()
ORDER BY (event_time, query_duration_ms)
AS
SELECT
    event_time,
    user,
    query_duration_ms,
    read_rows,
    memory_usage,
    query
FROM system.query_log
WHERE
    type = 'QueryFinish'
    AND query_duration_ms > 5000;
```

## Summary

ClickHouse's `system.query_log` provides rich execution metadata that makes slow query tracking straightforward. By filtering on `query_duration_ms`, grouping by normalized query hash, and monitoring memory and row reads, you can quickly identify and prioritize query optimizations. Pair this with materialized views or external alerting tools for continuous monitoring.
