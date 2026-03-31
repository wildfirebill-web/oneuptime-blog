# How to Understand InnoDB Buffer Pool Prefetching in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, InnoDB, Buffer Pool, Read-Ahead, Performance

Description: Learn how InnoDB buffer pool read-ahead (prefetching) works, when it helps, and how to monitor and tune it for sequential and random workloads.

---

## What Is InnoDB Prefetching?

When MySQL reads a page from disk into the buffer pool, InnoDB can predict that nearby pages will also be needed soon. It then reads those pages into the buffer pool proactively - before a query asks for them. This is called read-ahead or prefetching. Effective prefetching reduces I/O latency by hiding disk read costs.

InnoDB implements two read-ahead algorithms:
- **Linear read-ahead**: triggers when sequential pages are accessed in order
- **Random read-ahead**: triggers when many pages from the same extent are in the buffer pool

## Linear Read-Ahead

Linear read-ahead activates when MySQL reads a consecutive sequence of pages from the same extent (64 pages = 1 MB by default). When the threshold number of pages in the sequence are accessed, InnoDB prefetches the entire next extent.

```sql
-- Number of sequential page accesses to trigger linear read-ahead
-- Default: 56. Set to 0 to disable linear read-ahead.
SHOW VARIABLES LIKE 'innodb_read_ahead_threshold';

-- Tune for your workload
SET GLOBAL innodb_read_ahead_threshold = 24;  -- more aggressive
```

Lower values trigger prefetch earlier (better for sequential full scans), but can waste I/O on random workloads.

## Random Read-Ahead

Random read-ahead triggers when a large portion of an extent's pages are already in the buffer pool, suggesting the workload will need the remaining pages soon. It is disabled by default in MySQL 5.5+ because it caused I/O storms on random workloads.

```sql
-- Enable random read-ahead (off by default)
SHOW VARIABLES LIKE 'innodb_random_read_ahead';
SET GLOBAL innodb_random_read_ahead = ON;
```

Only enable this if your workload performs repeated access to the same data regions (e.g., hot tables repeatedly scanned).

## Monitoring Prefetch Effectiveness

```sql
-- Check read-ahead activity
SELECT name, count
FROM information_schema.innodb_metrics
WHERE name IN (
  'buffer_read_ahead',          -- pages prefetched by read-ahead
  'buffer_read_ahead_evicted',  -- prefetched pages evicted without being used
  'buffer_read_ahead_random'    -- pages prefetched by random read-ahead
);
```

A high `buffer_read_ahead_evicted` relative to `buffer_read_ahead` means prefetching is wasteful - pages are being read but not used before eviction. Consider raising `innodb_read_ahead_threshold` to make prefetch less aggressive.

## Buffer Pool Size and Prefetch Interaction

Prefetching only helps if the buffer pool is large enough to hold prefetched pages until they are accessed. If the buffer pool is too small, prefetched pages are evicted by new reads before queries use them:

```sql
-- Check buffer pool hit rate
SELECT
  (1 - (Innodb_buffer_pool_reads / Innodb_buffer_pool_read_requests)) * 100
    AS hit_rate_percent
FROM (
  SELECT variable_value AS Innodb_buffer_pool_reads
  FROM information_schema.global_status
  WHERE variable_name = 'Innodb_buffer_pool_reads'
) r,
(
  SELECT variable_value AS Innodb_buffer_pool_read_requests
  FROM information_schema.global_status
  WHERE variable_name = 'Innodb_buffer_pool_read_requests'
) rr;
```

A hit rate below 95% suggests the buffer pool is too small or prefetch is evicting useful data.

## Read-Ahead for Full Table Scans

Full table scans trigger aggressive linear read-ahead automatically. To avoid polluting the buffer pool with scan data that won't be reused, InnoDB uses a separate "young" and "old" list structure. Pages brought in by scans start in the "old" sublist and are only promoted to "young" if accessed again:

```sql
SHOW VARIABLES LIKE 'innodb_old_blocks_pct';       -- default 37% for old sublist
SHOW VARIABLES LIKE 'innodb_old_blocks_time';      -- ms a page must stay in old before promotion
```

Increase `innodb_old_blocks_time` (e.g., to 1000 ms) to protect frequently-accessed ("hot") data from being evicted by large scan operations.

## Summary

InnoDB prefetching uses linear and random read-ahead to bring disk pages into the buffer pool before queries request them. Linear read-ahead is enabled by default and tunable via `innodb_read_ahead_threshold`. Monitor `buffer_read_ahead_evicted` to detect wasteful prefetching, and ensure the buffer pool is large enough that prefetched pages remain until used. For workloads mixing scans with point lookups, tune `innodb_old_blocks_time` to protect hot data from scan-driven eviction.
