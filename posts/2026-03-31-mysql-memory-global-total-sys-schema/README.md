# How to Use the memory_global_total View in MySQL sys Schema

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, sys Schema, Memory, Performance Schema, Monitoring

Description: Use the MySQL sys schema memory views to monitor total server memory usage, per-user allocation, and per-component memory consumption.

---

## Overview

Memory management is often overlooked until MySQL crashes with an out-of-memory error. The `sys` schema provides memory tracking views built on Performance Schema's memory instrumentation, allowing you to see exactly how much memory MySQL is using and where it is allocated.

## Using memory_global_total

The `memory_global_total` view shows the total memory currently allocated across the MySQL server:

```sql
SELECT *
FROM sys.memory_global_total;
```

Output:
```text
+-----------------+
| total_allocated |
+-----------------+
| 2.28 GiB        |
+-----------------+
```

## Enabling Memory Instrumentation

```sql
UPDATE performance_schema.setup_instruments
SET ENABLED = 'YES'
WHERE NAME LIKE 'memory/%';
```

## Per-Component Memory Breakdown with memory_global_by_current_bytes

```sql
SELECT
  event_name,
  current_count,
  current_alloc,
  current_avg_alloc,
  high_count,
  high_alloc
FROM sys.memory_global_by_current_bytes
ORDER BY current_alloc DESC
LIMIT 15;
```

This shows which MySQL subsystem is using the most memory right now.

## Per-User Memory Usage

```sql
SELECT
  user,
  current_allocated,
  total_allocated
FROM sys.memory_by_user_by_current_bytes
ORDER BY current_allocated DESC;
```

## Per-Host Memory Usage

```sql
SELECT
  host,
  current_allocated,
  total_allocated
FROM sys.memory_by_host_by_current_bytes
ORDER BY current_allocated DESC;
```

## Per-Thread Memory Usage

```sql
SELECT
  thread_id,
  user,
  current_allocated,
  total_allocated
FROM sys.memory_by_thread_by_current_bytes
ORDER BY current_allocated DESC
LIMIT 10;
```

## Querying Raw Performance Schema Memory Tables

```sql
SELECT
  EVENT_NAME,
  CURRENT_COUNT_USED,
  CURRENT_NUMBER_OF_BYTES_USED / 1024 / 1024 AS current_mb,
  HIGH_NUMBER_OF_BYTES_USED / 1024 / 1024 AS peak_mb
FROM performance_schema.memory_summary_global_by_event_name
WHERE CURRENT_NUMBER_OF_BYTES_USED > 1048576
ORDER BY CURRENT_NUMBER_OF_BYTES_USED DESC
LIMIT 15;
```

## Identifying Memory Leaks

Compare current allocation to peak allocation. If a component's peak far exceeds its current allocation, a memory leak may have occurred and been freed, or the workload was much heavier in the past:

```sql
SELECT
  event_name,
  current_alloc,
  high_alloc
FROM sys.memory_global_by_current_bytes
WHERE current_alloc != high_alloc
ORDER BY high_alloc DESC
LIMIT 10;
```

## Summary

The `sys.memory_global_total` view and the `memory_global_by_current_bytes`, `memory_by_user_by_current_bytes`, and `memory_by_thread_by_current_bytes` views give you a complete picture of MySQL memory consumption. Use these to identify memory-hungry components, detect unusual growth patterns, and right-size MySQL memory parameters like `innodb_buffer_pool_size`, `sort_buffer_size`, and `join_buffer_size`.
