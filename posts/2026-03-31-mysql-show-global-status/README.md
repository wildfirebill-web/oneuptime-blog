# How to Use SHOW GLOBAL STATUS in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Monitoring, Status Variable, Performance, Administration

Description: Learn how to use SHOW GLOBAL STATUS in MySQL to monitor server-wide metrics including connections, queries, cache hits, and InnoDB performance counters.

---

`SHOW GLOBAL STATUS` returns server-wide cumulative counters that measure MySQL performance since the last restart. Unlike `SHOW STATUS`, which can return per-session values, `SHOW GLOBAL STATUS` always reflects the aggregate across all connections. It is the foundation of MySQL monitoring.

## Basic Usage

```sql
SHOW GLOBAL STATUS;
```

This returns all available status variables. The output typically contains 400+ rows in MySQL 8.0.

To filter by pattern:

```sql
SHOW GLOBAL STATUS LIKE 'Threads%';
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool%';
SHOW GLOBAL STATUS LIKE 'Com_select';
```

## Key Metrics to Monitor

### Connection Metrics

```sql
SHOW GLOBAL STATUS LIKE 'Connections';
SHOW GLOBAL STATUS LIKE 'Max_used_connections';
SHOW GLOBAL STATUS LIKE 'Threads_connected';
SHOW GLOBAL STATUS LIKE 'Threads_running';
SHOW GLOBAL STATUS LIKE 'Connection_errors_max_connections';
```

`Max_used_connections` shows the historical peak - compare it to `max_connections` to see how close you have been to the limit.

### Query Volume

```sql
SHOW GLOBAL STATUS LIKE 'Questions';
SHOW GLOBAL STATUS LIKE 'Com_select';
SHOW GLOBAL STATUS LIKE 'Com_insert';
SHOW GLOBAL STATUS LIKE 'Com_update';
SHOW GLOBAL STATUS LIKE 'Com_delete';
SHOW GLOBAL STATUS LIKE 'Slow_queries';
```

### InnoDB Buffer Pool

```sql
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_read_requests';
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_reads';
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_pages_data';
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_pages_free';
```

Calculate the buffer pool hit rate:

```sql
SELECT
  (1 - (
    (SELECT VARIABLE_VALUE FROM performance_schema.global_status WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads') /
    (SELECT VARIABLE_VALUE FROM performance_schema.global_status WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests')
  )) * 100 AS buffer_pool_hit_rate_pct;
```

A hit rate above 99% is healthy. Below 95% suggests the buffer pool is too small.

### Temporary Tables

```sql
SHOW GLOBAL STATUS LIKE 'Created_tmp_disk_tables';
SHOW GLOBAL STATUS LIKE 'Created_tmp_tables';
```

A high ratio of disk to memory temporary tables indicates queries are creating large temporary datasets. Consider increasing `tmp_table_size` and `max_heap_table_size`.

## Comparing Snapshots for Rate Calculation

Because status variables are cumulative, calculate rates by taking two snapshots separated by a time interval:

```sql
-- Snapshot 1
SELECT VARIABLE_VALUE INTO @q1
FROM performance_schema.global_status
WHERE VARIABLE_NAME = 'Questions';

-- Wait 60 seconds, then:
SELECT VARIABLE_VALUE INTO @q2
FROM performance_schema.global_status
WHERE VARIABLE_NAME = 'Questions';

SELECT (@q2 - @q1) / 60 AS queries_per_second;
```

## Resetting Status Counters

You can reset non-cumulative status variables without restarting the server:

```sql
FLUSH STATUS;
```

This resets most per-session and many global status variables, which is useful when you want to measure a specific workload period from a clean baseline.

## Querying via performance_schema

In MySQL 5.7 and later, you can also query status variables from the Performance Schema:

```sql
SELECT VARIABLE_NAME, VARIABLE_VALUE
FROM performance_schema.global_status
WHERE VARIABLE_NAME IN ('Questions', 'Slow_queries', 'Threads_connected')
ORDER BY VARIABLE_NAME;
```

## Summary

`SHOW GLOBAL STATUS` exposes the cumulative metrics that define MySQL server health. Focus on connection peaks, query volumes, InnoDB buffer pool hit rates, and temporary table spill rates. Calculate rates by comparing snapshots over time rather than reading raw cumulative values, and use `FLUSH STATUS` to reset counters when benchmarking specific workloads.
