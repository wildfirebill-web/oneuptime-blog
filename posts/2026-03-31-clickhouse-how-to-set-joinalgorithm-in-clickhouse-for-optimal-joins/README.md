# How to Set join_algorithm in ClickHouse for Optimal Joins

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, join_algorithm, Joins, Performance, SQL

Description: Learn how to configure the join_algorithm setting in ClickHouse to choose between hash join, merge join, partial merge, and other strategies for optimal query performance.

---

## Overview

ClickHouse supports multiple join algorithms, each with different performance characteristics depending on table sizes, data ordering, and available memory. The `join_algorithm` setting selects which algorithm to use. Choosing the right algorithm can dramatically reduce query latency and memory usage.

## Available Join Algorithms

```sql
SELECT getSetting('join_algorithm') AS default_algorithm
-- 'default' (which selects hash join by default)
```

Available values:
- `hash` - in-memory hash join (default for small-to-medium right tables)
- `partial_merge` - external merge join (for right tables that exceed memory)
- `prefer_partial_merge` - hash if fits in memory, else partial_merge
- `parallel_hash` - parallelized hash join (faster for large joins with sufficient RAM)
- `full_sorting_merge` - requires both sides sorted on join key
- `grace_hash` - Grace Hash join (spills to disk gracefully)
- `auto` - automatically selects based on table size

## Hash Join - Default

Hash join builds a hash table from the right table in memory, then probes it for each row of the left table. Best when the right table fits in memory.

```sql
SELECT a.user_id, a.event, b.name
FROM events a
JOIN users b ON a.user_id = b.user_id
SETTINGS join_algorithm = 'hash';
```

Control memory for hash join:

```sql
SETTINGS
    join_algorithm = 'hash',
    max_bytes_in_join = 1073741824;  -- 1 GiB
```

## Partial Merge Join

Partial merge join sorts and merges data externally, suitable for right tables that exceed memory:

```sql
SELECT a.order_id, b.product_name
FROM orders a
JOIN large_product_catalog b ON a.product_id = b.product_id
SETTINGS join_algorithm = 'partial_merge';
```

Slower than hash join but handles arbitrarily large joins without OOM errors.

## Parallel Hash Join

Uses multiple threads to build the hash table - faster for large right tables when memory allows:

```sql
SELECT a.session_id, b.page_path
FROM sessions a
JOIN pages b ON a.page_id = b.page_id
SETTINGS
    join_algorithm = 'parallel_hash',
    max_threads     = 16;
```

## Grace Hash Join

Grace Hash join partitions both sides to disk when the right table is too large for memory, then processes each partition pair:

```sql
SELECT a.event_id, b.campaign_name
FROM events a
JOIN large_campaigns b ON a.campaign_id = b.campaign_id
SETTINGS join_algorithm = 'grace_hash';
```

## Full Sorting Merge Join

Requires both sides of the join to be sorted on the join key. Most efficient when data is already sorted in storage:

```sql
SELECT a.user_id, b.profile
FROM sorted_events a
JOIN sorted_users b ON a.user_id = b.user_id
SETTINGS join_algorithm = 'full_sorting_merge';
```

## Auto Selection

`auto` lets ClickHouse choose based on the estimated size of the right table:

```sql
SET join_algorithm = 'auto';
```

ClickHouse starts with hash join and switches to grace_hash if the right table exceeds `max_bytes_in_join`.

## Prefer Partial Merge

A common production setting that uses hash join when the right table fits in memory, otherwise falls back to partial merge:

```sql
SET join_algorithm = 'prefer_partial_merge';
```

## Checking Available Memory for Joins

```sql
SELECT
    query_id,
    peak_memory_usage,
    query
FROM system.query_log
WHERE query LIKE '%JOIN%'
  AND type = 'QueryFinish'
ORDER BY event_time DESC
LIMIT 10
```

## Decision Guide

| Right Table Size | Recommended Algorithm |
|-----------------|----------------------|
| Small (< 1 GB)  | hash |
| Medium (1-10 GB) | parallel_hash or prefer_partial_merge |
| Large (> 10 GB) | grace_hash or partial_merge |
| Both sides sorted | full_sorting_merge |
| Unknown / mixed | auto |

## Memory Limits

```sql
SETTINGS
    join_algorithm = 'hash',
    max_bytes_in_join = 2147483648,  -- 2 GiB
    join_overflow_mode = 'throw';    -- or 'break' to return partial results
```

## Summary

ClickHouse's `join_algorithm` setting controls how JOIN operations are executed. Use `hash` for small right tables in memory, `parallel_hash` for faster large in-memory joins, `grace_hash` or `partial_merge` for right tables exceeding available RAM, and `auto` or `prefer_partial_merge` for workloads with variable table sizes. Always set `max_bytes_in_join` to prevent OOM errors on unexpected large inputs.
