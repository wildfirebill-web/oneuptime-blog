# How to Use SHOW PROCESSLIST in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, SHOW, PROCESSLIST, Monitoring, Active Queries

Description: Learn how to use SHOW PROCESSLIST and system.processes in ClickHouse to monitor active queries, memory usage, and execution time.

---

`SHOW PROCESSLIST` in ClickHouse returns a snapshot of all queries currently executing on the server. It is the fastest way to see what is running right now, who is running it, how long it has been running, and how much memory it is consuming. For deeper filtering and analysis, `system.processes` exposes the same data in a fully queryable form.

## Basic SHOW PROCESSLIST Syntax

```sql
SHOW PROCESSLIST;
```

Sample output (abbreviated):

```text
query_id                             | user    | elapsed | memory_usage | query
a1b2c3d4-0000-0000-0000-000000000001 | analyst | 12.34   | 524288000    | SELECT region, count() FROM events GROUP BY region
e5f6a7b8-0000-0000-0000-000000000002 | default | 0.05    | 1048576      | SHOW PROCESSLIST
```

Key columns:
- `query_id` - unique identifier for the query, used with KILL QUERY
- `user` - the user who submitted the query
- `elapsed` - seconds since the query started
- `memory_usage` - bytes of memory currently allocated
- `query` - the SQL text (may be truncated)

## Querying system.processes for Full Details

`system.processes` provides all the same columns plus many additional ones:

```sql
SELECT
    query_id,
    user,
    elapsed,
    formatReadableSize(memory_usage) AS memory,
    formatReadableSize(read_bytes)   AS bytes_read,
    read_rows,
    query
FROM system.processes
ORDER BY elapsed DESC;
```

```text
query_id       | user    | elapsed | memory    | bytes_read | read_rows | query
a1b2c3d4-...   | analyst | 45.12   | 512.00 MiB| 2.34 GiB  | 98432000  | SELECT region...
b9c0d1e2-...   | etl     | 3.02    | 12.50 MiB | 128.00 MiB| 4096000   | INSERT INTO...
```

## Monitoring Long-Running Queries

Find queries that have been running longer than 30 seconds:

```sql
SELECT
    query_id,
    user,
    round(elapsed, 2)                AS elapsed_sec,
    formatReadableSize(memory_usage) AS memory,
    left(query, 120)                 AS query_preview
FROM system.processes
WHERE elapsed > 30
ORDER BY elapsed DESC;
```

## Monitoring Memory-Intensive Queries

Identify which queries are consuming the most memory:

```sql
SELECT
    query_id,
    user,
    formatReadableSize(memory_usage)   AS memory,
    formatReadableSize(peak_memory_usage) AS peak_memory,
    round(elapsed, 2)                  AS elapsed_sec,
    left(query, 100)                   AS query_preview
FROM system.processes
ORDER BY memory_usage DESC
LIMIT 10;
```

## Viewing Active Queries by User

```sql
SELECT
    user,
    count()                               AS active_queries,
    formatReadableSize(sum(memory_usage)) AS total_memory,
    max(elapsed)                          AS longest_elapsed
FROM system.processes
GROUP BY user
ORDER BY active_queries DESC;
```

This gives a per-user summary of server load, useful for capacity planning or identifying runaway users.

## Checking Read Progress for Long Queries

For long-running SELECT queries you can monitor read progress:

```sql
SELECT
    query_id,
    user,
    round(elapsed, 2)                                          AS elapsed_sec,
    read_rows,
    total_rows_approx,
    if(total_rows_approx > 0,
       round(read_rows / total_rows_approx * 100, 1),
       NULL)                                                   AS progress_pct,
    formatReadableSize(memory_usage)                           AS memory
FROM system.processes
WHERE total_rows_approx > 0
ORDER BY elapsed DESC;
```

```text
query_id   | elapsed_sec | read_rows | total_rows_approx | progress_pct | memory
a1b2c3d4   | 45.12       | 48000000  | 100000000         | 48.0         | 512.00 MiB
```

## Watching a Specific Query

To watch a specific query's progress over time, run this in a loop from the shell:

```bash
watch -n 2 "clickhouse-client --query \"
SELECT
    query_id,
    round(elapsed, 2) AS elapsed_sec,
    read_rows,
    formatReadableSize(memory_usage) AS memory
FROM system.processes
WHERE query_id = 'a1b2c3d4-0000-0000-0000-000000000001'
\""
```

## Summary

`SHOW PROCESSLIST` and `system.processes` both expose the set of currently running queries in ClickHouse. `SHOW PROCESSLIST` is convenient for a quick interactive check, while `system.processes` supports full SQL filtering, aggregation, and joins. Monitor `elapsed`, `memory_usage`, and `read_rows` to identify runaway queries early, and capture `query_id` values before using KILL QUERY to terminate a problem query.
