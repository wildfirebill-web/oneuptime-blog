# How to Diagnose ClickHouse CPU Spikes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, CPU, Diagnosis, Performance, Query Profiler, Flamegraph

Description: Diagnose ClickHouse CPU spikes by identifying runaway queries, background merge activity, and code-level hotspots using system tables and the sampling profiler.

---

## What Causes CPU Spikes?

CPU spikes in ClickHouse typically come from:
- Analytical queries scanning billions of rows
- Complex expression evaluation (regex, string operations)
- Hash table construction for large GROUP BY or JOIN
- Background merges on large parts
- Codec decompression (especially ZSTD on large columns)

## Step 1 - Check Live CPU Consumers

```sql
SELECT
    query_id,
    user,
    elapsed,
    formatReadableSize(memory_usage) AS mem,
    ProfileEvents['OSCPUVirtualTimeMicroseconds'] / 1e6 AS cpu_sec,
    query
FROM system.processes
ORDER BY cpu_sec DESC;
```

## Step 2 - Recent High-CPU Queries

```sql
SELECT
    query_id,
    user,
    query_duration_ms,
    ProfileEvents['OSCPUVirtualTimeMicroseconds'] / 1e6 AS cpu_sec,
    read_rows,
    query
FROM system.query_log
WHERE event_date = today()
  AND type = 'QueryFinish'
ORDER BY cpu_sec DESC
LIMIT 20;
```

## Step 3 - Check Background Merge CPU

Merges run in the background and can consume significant CPU. Monitor their activity:

```sql
SELECT
    table,
    progress,
    elapsed,
    rows_read,
    rows_written
FROM system.merges
ORDER BY elapsed DESC;
```

During a ZSTD recompression merge, each background thread uses a full CPU core.

## Step 4 - Use the Query Profiler

Enable CPU profiling for a specific query:

```sql
SET query_profiler_cpu_time_period_ns = 10000000;

SELECT
    user_id,
    groupArray(event_type) AS events
FROM raw_events
WHERE ts >= today()
GROUP BY user_id;
```

Read the top stack frames:

```sql
SELECT
    count() AS samples,
    arrayStringConcat(
        arrayMap(x -> demangle(addressToSymbol(x)), trace),
        '\n'
    ) AS stack
FROM system.trace_log
WHERE trace_type = 'CPU'
  AND event_date = today()
GROUP BY trace
ORDER BY samples DESC
LIMIT 10;
```

## Killing Runaway Queries

```sql
KILL QUERY WHERE elapsed > 120;
-- Or target a specific user
KILL QUERY WHERE user = 'bad_user' AND elapsed > 30;
```

## Reducing CPU Usage

For regex-heavy queries, replace `match()` with indexed alternatives:

```sql
-- Expensive
SELECT * FROM logs WHERE match(message, 'error|fail|exception');

-- Better with bloom filter index
SELECT * FROM logs WHERE hasToken(message, 'error');
```

For GROUP BY hotspots, pre-aggregate with materialized views to reduce runtime computation.

## Summary

Diagnose ClickHouse CPU spikes by inspecting `system.processes` for live high-CPU queries, querying `system.query_log` for recent CPU consumers, checking `system.merges` for background merge pressure, and using the sampling profiler to identify code-level hotspots. Kill runaway queries and optimize the worst offenders with pre-aggregation or better index usage.
