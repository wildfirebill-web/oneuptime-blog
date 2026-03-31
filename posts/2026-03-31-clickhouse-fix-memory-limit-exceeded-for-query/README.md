# How to Fix "Memory limit exceeded for query" in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Memory, Error, Query Optimization, Troubleshooting

Description: Fix "Memory limit exceeded for query" errors in ClickHouse by tuning memory limits, enabling external aggregation, and optimizing query structure.

---

ClickHouse enforces per-query memory limits to protect server stability. When a query exceeds `max_memory_usage`, you get: `DB::Exception: Memory limit (for query) exceeded`. The fix is either raising the limit, enabling disk spilling, or rewriting the query to use less memory.

## Check Current Memory Limits

```sql
SELECT name, value
FROM system.settings
WHERE name IN (
    'max_memory_usage',
    'max_memory_usage_for_all_queries',
    'max_bytes_before_external_group_by',
    'max_bytes_before_external_sort'
);
```

## Increase the Per-Query Limit

For a single query session:

```sql
SET max_memory_usage = 20000000000;  -- 20 GB
SELECT count(), sum(value) FROM large_table GROUP BY category;
```

In `users.xml` for a specific user profile:

```xml
<profiles>
  <analyst>
    <max_memory_usage>30000000000</max_memory_usage>
  </analyst>
</profiles>
```

## Enable External Aggregation

Allow GROUP BY to spill to disk when memory is exhausted:

```sql
SET max_bytes_before_external_group_by = 10000000000;  -- 10 GB threshold
SET max_memory_usage = 20000000000;

SELECT user_id, count(), sum(revenue)
FROM events
GROUP BY user_id;
```

## Enable External Sorting

For ORDER BY queries that exceed memory:

```sql
SET max_bytes_before_external_sort = 10000000000;

SELECT * FROM large_table
ORDER BY timestamp DESC
LIMIT 1000000;
```

## Reduce Memory with Query Rewriting

Avoid loading full datasets into memory:

```sql
-- Instead of loading all data then filtering:
SELECT * FROM events WHERE user_id IN (
    SELECT user_id FROM users WHERE premium = 1
);

-- Use a JOIN with a small table on the right side:
SELECT e.*
FROM events e
INNER JOIN (
    SELECT user_id FROM users WHERE premium = 1
) u ON e.user_id = u.user_id;
```

## Use Sampling for Approximate Results

```sql
SELECT
    count() * 10 AS approx_count,
    avg(value) AS avg_value
FROM large_table SAMPLE 0.1  -- 10% sample
WHERE event_date = today();
```

## Monitor Active Query Memory

```sql
SELECT
    query_id,
    formatReadableSize(memory_usage) AS mem,
    elapsed,
    query
FROM system.processes
ORDER BY memory_usage DESC;
```

## Summary

"Memory limit exceeded for query" errors require a tiered approach: raise `max_memory_usage` for legitimate large queries, enable `max_bytes_before_external_group_by` and `max_bytes_before_external_sort` to spill to disk, and rewrite queries to avoid full-table scans. Use sampling for approximate analytics that do not require exact results.
