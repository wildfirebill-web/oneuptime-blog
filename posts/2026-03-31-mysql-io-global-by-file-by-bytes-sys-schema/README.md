# How to Use the io_global_by_file_by_bytes View in MySQL sys Schema

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, sys Schema, File I/O, InnoDB, Performance

Description: Use the MySQL sys schema io_global_by_file_by_bytes view to identify which database files generate the most read and write I/O traffic.

---

## Overview

The `sys.io_global_by_file_by_bytes` view shows global file I/O statistics sorted by total bytes transferred. It is the fastest way to identify which InnoDB tablespace files, redo log files, or binary logs are generating the most disk traffic.

## Basic Query

```sql
SELECT *
FROM sys.io_global_by_file_by_bytes
LIMIT 10;
```

## Key Columns

```sql
SELECT
  file,
  count_read,
  total_read,
  avg_read,
  count_write,
  total_written,
  avg_write,
  total,
  write_pct
FROM sys.io_global_by_file_by_bytes
ORDER BY total DESC
LIMIT 15;
```

- `file` - file path (shortened for readability)
- `count_read` / `count_write` - number of read/write operations
- `total_read` / `total_written` - formatted bytes transferred
- `avg_read` / `avg_write` - average bytes per operation
- `total` - combined bytes
- `write_pct` - percentage of total I/O that is writes

## Identifying Write-Heavy Files

High write percentages on `.ibd` files indicate heavy DML workloads. High writes on `ib_logfile` files indicate redo log pressure:

```sql
SELECT
  file,
  count_write,
  total_written,
  write_pct
FROM sys.io_global_by_file_by_bytes
WHERE write_pct > 70
ORDER BY total_written DESC;
```

## Finding Read-Heavy Tablespaces

High reads on specific `.ibd` files may indicate those tables need better caching:

```sql
SELECT
  file,
  count_read,
  total_read,
  avg_read
FROM sys.io_global_by_file_by_bytes
WHERE file LIKE '%.ibd'
ORDER BY count_read DESC
LIMIT 10;
```

## Using the Companion View by Latency

```sql
-- Sort by latency instead of bytes
SELECT
  file,
  total_latency,
  count_read,
  read_latency,
  count_write,
  write_latency
FROM sys.io_global_by_file_by_latency
ORDER BY total_latency DESC
LIMIT 10;
```

## Querying the Raw Performance Schema Table

```sql
SELECT
  FILE_NAME,
  COUNT_READ,
  SUM_NUMBER_OF_BYTES_READ AS bytes_read,
  COUNT_WRITE,
  SUM_NUMBER_OF_BYTES_WRITE AS bytes_written,
  (SUM_NUMBER_OF_BYTES_READ + SUM_NUMBER_OF_BYTES_WRITE) AS total_bytes
FROM performance_schema.file_summary_by_instance
ORDER BY total_bytes DESC
LIMIT 10;
```

## Practical Use Cases

1. **Binary log I/O**: If binlog files dominate the write I/O, tune `sync_binlog` and `binlog_cache_size`
2. **Redo log pressure**: High writes on `ib_logfile*` suggest increasing `innodb_log_file_size`
3. **Hot tablespace**: A single `.ibd` file with extreme read volume may need its table partitioned or split
4. **Temp file I/O**: High activity on `#sql-*.ibd` temp files indicates queries creating large temporary tables on disk

## Summary

The `sys.io_global_by_file_by_bytes` view translates raw Performance Schema file I/O counters into actionable insights. By identifying which files generate the most read and write traffic, you can tune InnoDB buffer pool size, redo log configuration, and binary log settings to reduce disk I/O bottlenecks.
