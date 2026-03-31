# How to Fix 'Memory limit exceeded for query' in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Troubleshooting, Memory, Query, Optimization

Description: Fix 'Memory limit exceeded for query' errors in ClickHouse by increasing query memory limits, enabling external aggregation, and optimizing high-memory queries.

---

The "Memory limit exceeded for query" error means a single query consumed more memory than allowed by `max_memory_usage`. This is distinct from the server-wide memory limit and applies per-query.

## Understanding the Error

```text
Code: 241. DB::Exception: Memory limit (for query) exceeded: would use X bytes (attempt to allocate chunk of Y bytes), maximum: Z bytes.
```

The limit is controlled by `max_memory_usage` in the user's profile settings.

## Check Current Limits

```sql
SELECT name, value
FROM system.settings
WHERE name IN (
    'max_memory_usage',
    'max_memory_usage_for_user',
    'max_server_memory_usage',
    'max_bytes_before_external_group_by',
    'max_bytes_before_external_sort'
);
```

## Fix 1: Increase max_memory_usage for the Query

Temporarily raise the limit for a specific query:

```sql
SET max_memory_usage = 64424509440; -- 60 GB
SELECT ... your heavy query ...;
```

Set it permanently in the user profile:

```xml
<clickhouse>
  <profiles>
    <analytics_user>
      <max_memory_usage>64424509440</max_memory_usage>
    </analytics_user>
  </profiles>
</clickhouse>
```

## Fix 2: Enable External Group By

Allow GROUP BY to spill to disk when memory is exhausted:

```sql
SET max_bytes_before_external_group_by = 10000000000; -- 10 GB threshold
SELECT user_id, count(), sum(revenue)
FROM events
GROUP BY user_id;
```

When the in-memory hash table grows beyond 10 GB, ClickHouse spills it to disk. This is slower but prevents the query from failing.

## Fix 3: Enable External Sort

Similarly for ORDER BY:

```sql
SET max_bytes_before_external_sort = 10000000000;
SELECT * FROM large_table ORDER BY ts DESC LIMIT 1000;
```

## Fix 4: Identify High-Memory Queries

Find which queries are using the most memory:

```sql
SELECT
    query_id,
    user,
    memory_usage,
    formatReadableSize(memory_usage) AS memory_readable,
    query
FROM system.processes
ORDER BY memory_usage DESC
LIMIT 10;
```

In query history:

```sql
SELECT
    query_id,
    user,
    memory_usage,
    formatReadableSize(memory_usage) AS mem,
    query
FROM system.query_log
WHERE type = 'QueryFinish'
  AND event_time > now() - INTERVAL 1 HOUR
ORDER BY memory_usage DESC
LIMIT 20;
```

## Fix 5: Optimize the Query

Reduce memory usage by optimizing the query structure:

```sql
-- Use PREWHERE to filter early (reduces rows loaded into memory)
SELECT user_id, count()
FROM events
PREWHERE event_date >= today() - 7
WHERE action = 'purchase'
GROUP BY user_id;

-- Avoid SELECT * - only select needed columns
SELECT user_id, action FROM events;  -- Not SELECT * FROM events

-- Use approximate aggregations when exact counts are not needed
SELECT uniq(user_id) FROM events;  -- Uses less memory than uniqExact
```

## Fix 6: Use query_memory_overcommit_numerator

Allow some memory overcommit:

```sql
SET memory_overcommit_ratio_denominator = 1073741824; -- 1 GB
SET memory_usage_overcommit_max_wait_microseconds = 5000000;
```

## Summary

"Memory limit exceeded for query" errors are fixed by increasing `max_memory_usage` for the user profile, enabling external group-by and sort with disk spillover via `max_bytes_before_external_group_by` and `max_bytes_before_external_sort`, and optimizing queries to use PREWHERE, select fewer columns, and use approximate functions like `uniq()` instead of `uniqExact()`.
