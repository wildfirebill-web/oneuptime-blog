# How to Monitor ClickHouse Memory Usage Per Query

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Memory, Monitoring, Query Performance, System Table

Description: Learn how to monitor and limit memory usage per query in ClickHouse using system tables, query settings, and memory profiling tools.

---

## Why Monitor Per-Query Memory

In a shared ClickHouse cluster, a single runaway query can exhaust available memory and cause other queries to fail. Understanding per-query memory consumption lets you set appropriate limits, identify expensive queries, and tune table design.

## Checking Active Query Memory Usage

```sql
SELECT
    query_id,
    user,
    elapsed,
    formatReadableSize(memory_usage) AS memory,
    formatReadableSize(peak_memory_usage) AS peak_memory,
    substring(query, 1, 100) AS query_snippet
FROM system.processes
ORDER BY memory_usage DESC;
```

## Historical Peak Memory from Query Log

```sql
SELECT
    query_id,
    user,
    toStartOfMinute(event_time) AS minute,
    formatReadableSize(memory_usage) AS memory_used,
    formatReadableSize(peak_memory_usage) AS peak_memory,
    query_duration_ms,
    substring(query, 1, 200) AS query_snippet
FROM system.query_log
WHERE type = 'QueryFinish'
  AND event_date >= today() - 1
ORDER BY peak_memory_usage DESC
LIMIT 20;
```

## Setting max_memory_usage Per Query

```sql
SELECT
    user_id,
    count() AS events,
    sum(amount) AS total
FROM orders
WHERE order_date >= today() - 90
GROUP BY user_id
SETTINGS max_memory_usage = 5000000000;  -- 5 GB limit
```

## Setting max_memory_usage in User Profiles

```xml
<profiles>
  <default>
    <max_memory_usage>10000000000</max_memory_usage>
    <max_memory_usage_for_all_queries>50000000000</max_memory_usage_for_all_queries>
  </default>
  <analyst>
    <max_memory_usage>50000000000</max_memory_usage>
  </analyst>
</profiles>
```

## Using External GROUP BY to Spill to Disk

Instead of failing when memory is exhausted, spill to disk:

```sql
SELECT
    user_id,
    count() AS events
FROM large_events
GROUP BY user_id
SETTINGS
    max_bytes_before_external_group_by = 10000000000,
    max_memory_usage = 20000000000;
```

## Using External Sort to Spill to Disk

```sql
SELECT *
FROM large_table
ORDER BY timestamp DESC
LIMIT 1000
SETTINGS
    max_bytes_before_external_sort = 10000000000;
```

## Aggregated Memory Report Per User

```sql
SELECT
    user,
    count() AS query_count,
    formatReadableSize(avg(peak_memory_usage)) AS avg_peak_memory,
    formatReadableSize(max(peak_memory_usage)) AS max_peak_memory,
    formatReadableSize(sum(peak_memory_usage)) AS total_peak_memory
FROM system.query_log
WHERE type = 'QueryFinish'
  AND event_date >= today() - 7
GROUP BY user
ORDER BY max_peak_memory DESC;
```

## Identifying Memory-Heavy Query Patterns

```sql
SELECT
    arrayStringConcat(tables, ', ') AS tables_used,
    count() AS query_count,
    formatReadableSize(avg(peak_memory_usage)) AS avg_peak_memory,
    formatReadableSize(max(peak_memory_usage)) AS max_peak_memory
FROM system.query_log
WHERE type = 'QueryFinish'
  AND event_date = today()
  AND peak_memory_usage > 1000000000  -- Over 1 GB
GROUP BY tables_used
ORDER BY max_peak_memory DESC
LIMIT 20;
```

## Killing a High-Memory Query

```sql
-- Get the query ID first
SELECT query_id, memory_usage, query
FROM system.processes
ORDER BY memory_usage DESC
LIMIT 5;

-- Kill the query
KILL QUERY WHERE query_id = 'your-query-id';
```

## Checking System-Wide Memory

```sql
SELECT
    metric,
    formatReadableSize(value) AS human_value,
    description
FROM system.asynchronous_metrics
WHERE metric LIKE '%Memory%'
ORDER BY value DESC;
```

## Setting Up Memory Alerts

Use Prometheus to alert when per-query memory exceeds thresholds:

```yaml
- alert: ClickHouseHighQueryMemory
  expr: ClickHouseMetrics_MemoryTracking > 20000000000
  for: 1m
  labels:
    severity: warning
  annotations:
    summary: "ClickHouse queries using over 20GB of memory"
```

## Summary

Monitor per-query memory in ClickHouse using `system.processes` for live data and `system.query_log` for historical peaks. Set `max_memory_usage` per query or per user profile to prevent runaway queries. For large GROUP BY and ORDER BY operations, configure external sort/group by spill-to-disk settings to handle data larger than available RAM gracefully.
