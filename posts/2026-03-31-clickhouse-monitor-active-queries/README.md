# How to Monitor and Manage Active Queries in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Active Query, Query Management, system.processes, Performance, Monitoring

Description: Learn how to monitor running queries, identify long-running operations, and kill stuck processes in ClickHouse using system tables.

---

ClickHouse provides built-in system tables to give you real-time visibility into what queries are currently executing. Whether you are debugging a slow dashboard or investigating resource saturation, understanding active queries is essential for operations.

## Viewing Active Queries

The `system.processes` table lists all queries currently running on the server:

```sql
SELECT
    query_id,
    user,
    elapsed,
    read_rows,
    memory_usage,
    query
FROM system.processes
ORDER BY elapsed DESC;
```

This shows you who is running what, how long it has been executing, and how much memory it is consuming.

## Filtering Long-Running Queries

To find queries that have been running for more than 30 seconds:

```sql
SELECT
    query_id,
    user,
    elapsed,
    formatReadableSize(memory_usage) AS mem,
    query
FROM system.processes
WHERE elapsed > 30
ORDER BY elapsed DESC;
```

## Killing a Stuck Query

If a query is consuming too many resources or appears hung, you can kill it using its `query_id`:

```sql
KILL QUERY WHERE query_id = 'abc123-xxxx-xxxx';
```

You can also target queries by user:

```sql
KILL QUERY WHERE user = 'analyst_user' AND elapsed > 120;
```

Use `KILL QUERY` with caution - it terminates the query immediately without a rollback mechanism.

## Checking Query Progress

For long-running queries, ClickHouse tracks progress:

```sql
SELECT
    query_id,
    elapsed,
    read_rows,
    total_rows_approx,
    round(100 * read_rows / total_rows_approx, 1) AS progress_pct
FROM system.processes
WHERE total_rows_approx > 0;
```

## Monitoring from the Query Log

After a query finishes, it moves to `system.query_log`. To analyze recent completions:

```sql
SELECT
    query_id,
    user,
    query_duration_ms,
    read_rows,
    memory_usage,
    exception
FROM system.query_log
WHERE event_time > now() - INTERVAL 10 MINUTE
  AND type = 'QueryFinish'
ORDER BY query_duration_ms DESC
LIMIT 20;
```

## Setting Query Timeout Limits

Prevent runaway queries by configuring max execution time per user or globally:

```sql
ALTER USER analyst_user SETTINGS max_execution_time = 60;
```

Or at the session level:

```sql
SET max_execution_time = 30;
SELECT count() FROM large_table;
```

## Summary

ClickHouse provides `system.processes` for real-time query visibility and `system.query_log` for historical analysis. Use `KILL QUERY` to terminate problematic queries, and enforce `max_execution_time` limits to protect your cluster from resource exhaustion.
