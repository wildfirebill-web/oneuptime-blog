# How to Fix "Sorting memory limit exceeded" in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Sorting, Memory, Error, Troubleshooting

Description: Fix "Sorting memory limit exceeded" errors in ClickHouse by enabling external sort, optimizing ORDER BY queries, and tuning memory settings.

---

ClickHouse raises "Sorting memory limit exceeded" when an ORDER BY operation requires more memory than the configured limit. Large result sets without a LIMIT clause or sorts on high-cardinality columns are the typical culprits.

## Identify the Query

Check currently running queries:

```sql
SELECT
    query_id,
    formatReadableSize(memory_usage) AS memory,
    elapsed,
    query
FROM system.processes
WHERE query ILIKE '%ORDER BY%'
ORDER BY memory_usage DESC;
```

## Enable External Sort (Spill to Disk)

The primary fix is to allow ClickHouse to spill sort data to disk:

```sql
SET max_bytes_before_external_sort = 10000000000;  -- 10 GB spill threshold
SET max_memory_usage = 20000000000;                -- overall query memory limit
```

With these settings, ClickHouse writes sorted chunks to temporary files when memory is tight:

```sql
SET max_bytes_before_external_sort = 10000000000;
SELECT user_id, event_time, event_type
FROM events
ORDER BY user_id, event_time
LIMIT 10000000;
```

## Set Persistently in users.xml

```xml
<profiles>
  <default>
    <max_bytes_before_external_sort>10000000000</max_bytes_before_external_sort>
    <max_memory_usage>20000000000</max_memory_usage>
  </default>
</profiles>
```

## Add LIMIT to Reduce Sort Size

If you only need the top-N rows, add a LIMIT - ClickHouse optimizes this into a partial sort:

```sql
-- Uses a heap instead of full sort, much less memory:
SELECT user_id, revenue
FROM orders
ORDER BY revenue DESC
LIMIT 1000;
```

## Use topK for Approximate Results

For approximate top-N without full sorting:

```sql
SELECT topK(100)(user_id) AS top_users
FROM events
WHERE event_date = today();
```

## Partition-Wise Processing

For large data sets, process partition by partition:

```sql
-- Process one month at a time
SELECT user_id, sum(revenue)
FROM orders
WHERE toYYYYMM(order_date) = 202401
GROUP BY user_id
ORDER BY sum(revenue) DESC
LIMIT 100;
```

## Optimize the Schema

If you frequently sort by the same column, include it in the ORDER BY clause of your MergeTree table definition - data will be pre-sorted on disk:

```sql
CREATE TABLE events (
    event_date Date,
    user_id UInt64,
    event_type String
) ENGINE = MergeTree()
ORDER BY (event_date, user_id);
```

Queries filtering and sorting by `event_date, user_id` will then read pre-sorted data.

## Summary

"Sorting memory limit exceeded" is resolved by enabling `max_bytes_before_external_sort` to spill to disk, adding LIMIT clauses to avoid full sorts, using approximate functions like `topK`, and designing MergeTree schemas with the sorting key matching query patterns. Process large sorts partition-by-partition to keep memory usage bounded.
