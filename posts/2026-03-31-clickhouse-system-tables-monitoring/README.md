# How to Use ClickHouse System Tables for Monitoring

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, System Table, Monitoring, Observability, Performance, Administration

Description: Learn how to use ClickHouse system tables including metrics, events, processes, replicas, and parts to monitor cluster health and diagnose performance issues.

---

ClickHouse ships with an extensive set of system tables that expose real-time and historical data about every aspect of the server: current metrics, cumulative event counts, active queries, replication state, background operations, and storage layout. Understanding these tables is the most direct way to diagnose problems without relying on external monitoring infrastructure.

## Overview of Key System Tables

```sql
-- List all system tables
SELECT name, comment
FROM system.tables
WHERE database = 'system'
ORDER BY name;
```

The most important tables for monitoring fall into five categories:

| Table | Purpose |
|---|---|
| `system.metrics` | Current snapshot of live metric gauges |
| `system.events` | Cumulative counters since server start |
| `system.asynchronous_metrics` | Background-computed metrics (memory, disk) |
| `system.processes` | Currently running queries |
| `system.query_log` | Historical query log |
| `system.replicas` | Replication status per table |
| `system.replication_queue` | Pending replication tasks |
| `system.parts` | Data part metadata for all MergeTree tables |
| `system.merges` | Active merge and mutation operations |
| `system.mutations` | Pending and completed mutations |
| `system.errors` | Recent error counts by error code |

## system.metrics - Live Gauges

`system.metrics` holds point-in-time values for active gauges such as current connections, in-flight queries, and active merges:

```sql
-- All current metrics
SELECT metric, value, description
FROM system.metrics
ORDER BY value DESC
LIMIT 30;

-- Connection counts
SELECT metric, value
FROM system.metrics
WHERE metric IN (
    'TCPConnection',
    'HTTPConnection',
    'MySQLConnection',
    'PostgreSQLConnection',
    'InterserverConnection'
);

-- Query and merge activity
SELECT metric, value
FROM system.metrics
WHERE metric IN (
    'Query',
    'Merge',
    'PartMutation',
    'ReplicatedFetch',
    'ReplicatedSend',
    'BackgroundPoolTask'
);
```

## system.events - Cumulative Counters

`system.events` stores monotonically increasing counters since server start. Use these to compute rates:

```sql
-- Top events by count
SELECT event, value, description
FROM system.events
WHERE value > 0
ORDER BY value DESC
LIMIT 20;

-- Compute rate since startup
SELECT
    event,
    value,
    uptime() AS server_uptime_seconds,
    round(value / uptime(), 2) AS per_second
FROM system.events
WHERE event IN (
    'Query',
    'SelectQuery',
    'InsertQuery',
    'MergeTreeDataWriterRows',
    'ReadCompressedBytes',
    'WriteBufferFromFileDescriptorWriteBytes'
)
ORDER BY per_second DESC;
```

## system.asynchronous_metrics - Background Metrics

These metrics are computed in a background thread and updated every few seconds. They include memory usage, disk space, and uptime:

```sql
-- Memory overview
SELECT metric, value
FROM system.asynchronous_metrics
WHERE metric LIKE '%Memory%'
ORDER BY metric;

-- Disk overview
SELECT
    metric,
    formatReadableSize(toUInt64(value)) AS human_readable,
    value
FROM system.asynchronous_metrics
WHERE metric LIKE '%Disk%'
ORDER BY metric;

-- Uptime and version
SELECT metric, value
FROM system.asynchronous_metrics
WHERE metric IN ('Uptime', 'NumberOfDatabases', 'NumberOfTables', 'TotalRowsOfMergeTreeTables');
```

## system.processes - Active Queries

Inspect currently executing queries in real time:

```sql
-- All running queries sorted by elapsed time
SELECT
    query_id,
    user,
    elapsed,
    formatReadableSize(read_bytes)   AS bytes_read,
    formatReadableSize(memory_usage) AS memory,
    read_rows,
    total_rows_approx,
    if(total_rows_approx > 0, round(read_rows / total_rows_approx * 100, 1), 0) AS pct_complete,
    query
FROM system.processes
ORDER BY elapsed DESC;

-- Show queries waiting for resources
SELECT query_id, user, elapsed, query
FROM system.processes
WHERE is_cancelled = 0 AND elapsed > 30
ORDER BY elapsed DESC;
```

## system.replicas - Replication Health

This is the most important table for replicated setups:

```sql
-- Full replication status
SELECT
    database,
    table,
    is_leader,
    is_readonly,
    is_session_expired,
    future_parts,
    parts_to_check,
    queue_size,
    inserts_in_queue,
    merges_in_queue,
    log_max_index,
    log_pointer,
    log_max_index - log_pointer AS replication_lag,
    last_queue_update,
    absolute_delay,
    zookeeper_path
FROM system.replicas
ORDER BY absolute_delay DESC;

-- Find tables with replication problems
SELECT database, table, is_readonly, queue_size, absolute_delay
FROM system.replicas
WHERE is_readonly = 1 OR absolute_delay > 60 OR queue_size > 100
ORDER BY absolute_delay DESC;
```

## system.replication_queue - Pending Tasks

When replication lag is high, drill into the queue to find what is blocked:

```sql
SELECT
    database,
    table,
    type,
    source_replica,
    new_part_name,
    create_time,
    num_tries,
    last_attempt_time,
    last_exception,
    is_currently_executing
FROM system.replication_queue
WHERE last_exception != ''
ORDER BY num_tries DESC
LIMIT 20;
```

## system.parts - Storage Layout

Understanding data part health is critical for MergeTree performance:

```sql
-- Part count and size per table
SELECT
    database,
    table,
    count()                                AS part_count,
    sum(rows)                             AS total_rows,
    formatReadableSize(sum(data_compressed_bytes))   AS compressed_size,
    formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed_size,
    round(sum(data_compressed_bytes) / sum(data_uncompressed_bytes) * 100, 1) AS compression_ratio_pct
FROM system.parts
WHERE active = 1
GROUP BY database, table
ORDER BY sum(data_compressed_bytes) DESC
LIMIT 20;

-- Tables with too many small parts (merge backlog indicator)
SELECT
    database,
    table,
    count() AS part_count
FROM system.parts
WHERE active = 1
GROUP BY database, table
HAVING part_count > 300
ORDER BY part_count DESC;
```

## system.merges - Active Merge Operations

```sql
SELECT
    database,
    table,
    elapsed,
    round(progress * 100, 1)                    AS pct_complete,
    formatReadableSize(total_size_bytes_compressed) AS total_size,
    formatReadableSize(bytes_read_uncompressed)     AS read_so_far,
    is_mutation,
    result_part_name
FROM system.merges
ORDER BY elapsed DESC;
```

## system.errors - Recent Errors

```sql
-- Most frequent errors
SELECT
    name,
    code,
    value,
    last_error_time,
    last_error_message
FROM system.errors
WHERE value > 0
ORDER BY value DESC
LIMIT 20;
```

## Building a Health Check Query

Combine multiple system tables into a single health summary:

```sql
SELECT
    (SELECT value FROM system.metrics WHERE metric = 'Query')            AS active_queries,
    (SELECT value FROM system.metrics WHERE metric = 'Merge')            AS active_merges,
    (SELECT value FROM system.metrics WHERE metric = 'TCPConnection')    AS tcp_connections,
    (SELECT formatReadableSize(toUInt64(value))
     FROM system.asynchronous_metrics WHERE metric = 'MemoryResident')  AS resident_memory,
    (SELECT max(absolute_delay) FROM system.replicas)                    AS max_replication_lag_s,
    (SELECT count() FROM system.replicas WHERE is_readonly = 1)         AS readonly_replicas,
    (SELECT count() FROM system.processes WHERE elapsed > 60)           AS long_running_queries;
```

## Summary

ClickHouse system tables are the primary diagnostic tool for operators. `system.metrics` and `system.asynchronous_metrics` provide instantaneous gauges for memory, disk, and concurrency. `system.events` exposes cumulative counters that you can use to compute per-second rates. `system.processes` reveals what is running right now. `system.replicas` and `system.replication_queue` expose replication state. `system.parts` reveals storage layout health. Together these tables give you complete, zero-dependency visibility into ClickHouse internals without requiring any external monitoring stack.
