# How to Use Memory Overcommit Manager in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Memory, Overcommit, Resource Management, Performance

Description: Learn how ClickHouse's memory overcommit manager allows queries to temporarily exceed memory limits under low pressure, improving throughput without risking OOM kills.

---

ClickHouse's memory overcommit manager is a smarter alternative to hard per-query memory limits. Instead of always enforcing a fixed ceiling, it allows queries to use more memory when the system has headroom, and begins canceling the most memory-hungry queries only when actual memory pressure occurs. This increases utilization during low-traffic periods without sacrificing stability during peaks.

## How Overcommit Works

Each query sets a "soft" memory limit. The overcommit manager tracks total server memory usage and maintains a ratio. As long as total memory usage is below the server's threshold, queries can exceed their soft limit up to a configurable multiplier. When the system approaches its hard limit, the overcommit manager picks the query consuming the most memory above its soft limit and cancels it with an exception.

## Configuring Overcommit

Set the user-level soft limit and overcommit ratio in the user profile:

```xml
<profiles>
  <default>
    <!-- Soft limit: 2 GB per query -->
    <max_memory_usage>2147483648</max_memory_usage>

    <!-- Allow up to 2x overcommit when system has headroom -->
    <memory_overcommit_ratio_denominator>1073741824</memory_overcommit_ratio_denominator>

    <!-- Same for user-level totals -->
    <memory_overcommit_ratio_denominator_for_user>1073741824</memory_overcommit_ratio_denominator_for_user>
  </default>
</profiles>
```

The server-level hard limit is set separately:

```xml
<max_server_memory_usage_to_ram_ratio>0.9</max_server_memory_usage_to_ram_ratio>
```

This limits ClickHouse to 90% of available RAM before the overcommit manager starts canceling queries.

## Session-Level Overcommit Settings

You can also configure overcommit at the session or query level:

```sql
-- Enable global memory overcommit tracking
SET global_memory_usage_overcommit_max_wait_microseconds = 200000;

-- Set per-query overcommit denominator (in bytes)
SET memory_overcommit_ratio_denominator = 1073741824;
```

The `global_memory_usage_overcommit_max_wait_microseconds` setting controls how long a query waits for other queries to free memory before it proceeds despite overcommit conditions.

## Monitoring Memory Overcommit Events

```sql
SELECT event, value
FROM system.events
WHERE event LIKE '%MemoryOvercommit%';
```

Key events:
- `MemoryOvercommitWaitTimeMicroseconds` - total time queries waited due to overcommit pressure
- `MemoryAllocatorPurge` - allocator releasing memory back to OS

Check current memory usage across running queries:

```sql
SELECT
    query_id,
    user,
    formatReadableSize(memory_usage) AS memory,
    substring(query, 1, 80) AS query_preview
FROM system.processes
ORDER BY memory_usage DESC
LIMIT 10;
```

## Identifying Queries Canceled by Overcommit

```sql
SELECT
    query_id,
    user,
    exception,
    formatReadableSize(memory_usage) AS peak_memory
FROM system.query_log
WHERE exception LIKE '%memory%'
    AND event_date = today()
ORDER BY event_time DESC
LIMIT 20;
```

Queries canceled by overcommit will show exceptions containing `Memory limit` or `overcommit`.

## When to Use Overcommit vs Hard Limits

| Scenario | Recommendation |
|---|---|
| Shared analytics cluster | Overcommit with server-level hard cap |
| Interactive dashboards | Hard limit with `break` overflow mode |
| Batch ETL jobs | Generous hard limit per job |
| Multi-tenant SaaS | Quotas combined with overcommit |

## Summary

The memory overcommit manager strikes a balance between resource utilization and stability. Queries can burst above their soft limit when memory is available, and the system gracefully cancels the most expensive query when pressure builds. Configure the `memory_overcommit_ratio_denominator` in user profiles, set a server-level RAM ratio cap, and monitor `MemoryOvercommitWaitTimeMicroseconds` to tune the balance for your workload.
