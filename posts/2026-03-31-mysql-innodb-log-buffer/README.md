# How to Configure InnoDB Log Buffer in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, InnoDB, Log Buffer, Performance, Configuration

Description: Learn how to configure the InnoDB log buffer size to reduce disk I/O for large or frequent transactions and improve write throughput in MySQL.

---

## Introduction

The InnoDB log buffer is an in-memory area that holds redo log entries before they are flushed to the redo log files on disk. Writing to memory is orders of magnitude faster than writing to disk, so a well-sized log buffer absorbs multiple small redo writes and flushes them in larger, more efficient batches.

The log buffer is controlled by `innodb_log_buffer_size` and is separate from both the buffer pool and the redo log files.

## How the log buffer fits into InnoDB write path

```mermaid
flowchart LR
    A[Transaction generates redo entries] --> B[InnoDB log buffer in RAM]
    B --> C{Flush trigger}
    C -->|Transaction commits innodb_flush_log_at_trx_commit=1| D[Flush + fsync to redo log files]
    C -->|Buffer 1/2 full| D
    C -->|1 second timer| D
    D --> E[ib_redo files on disk]
```

## Default and recommended values

| Workload | Recommended `innodb_log_buffer_size` |
|---|---|
| Low write volume | 16 MB (default) |
| Medium OLTP | 64 MB |
| High write volume / large transactions | 128 MB - 512 MB |
| Bulk inserts / data loading | 512 MB - 1 GB (temporarily) |

The default is 16 MB in MySQL 8.0, which is adequate for many workloads. Increase it when you see frequent log buffer flushes between commits.

## Checking the current setting

```sql
SHOW VARIABLES LIKE 'innodb_log_buffer_size';
/*
+------------------------+----------+
| Variable_name          | Value    |
+------------------------+----------+
| innodb_log_buffer_size | 16777216 |
+------------------------+----------+
*/

-- Convert to MB
SELECT @@innodb_log_buffer_size / 1024 / 1024 AS log_buffer_mb;
```

## Configuring the log buffer

`innodb_log_buffer_size` is dynamic in MySQL 8.0 -- it can be changed without a restart:

```sql
-- Dynamic change (MySQL 8.0)
SET GLOBAL innodb_log_buffer_size = 134217728; -- 128 MB

-- Verify
SELECT @@innodb_log_buffer_size / 1024 / 1024 AS log_buffer_mb;
```

For persistence, add to `my.cnf`:

```ini
# /etc/mysql/mysql.conf.d/mysqld.cnf

[mysqld]
innodb_log_buffer_size = 134217728   # 128 MB
```

## Detecting log buffer pressure

```sql
-- Check how often the log buffer is flushed because it is almost full
SHOW STATUS LIKE 'Innodb_log_waits';
-- Non-zero value means transactions are waiting for the log buffer to be flushed
-- A high or growing value indicates the buffer is too small

-- Check total log writes
SHOW STATUS LIKE 'Innodb_log_writes';

-- Ratio: waits / writes > 1% suggests increasing buffer size
SELECT
  (SELECT variable_value FROM performance_schema.global_status WHERE variable_name = 'Innodb_log_waits')  AS log_waits,
  (SELECT variable_value FROM performance_schema.global_status WHERE variable_name = 'Innodb_log_writes') AS log_writes;
```

## Flushing behavior controlled by innodb_flush_log_at_trx_commit

The log buffer flush policy interacts with `innodb_flush_log_at_trx_commit`:

| Setting | Flush behavior | Durability | Performance |
|---|---|---|---|
| `1` (default) | Flush + fsync on every COMMIT | Full ACID | Lowest write TPS |
| `2` | Write to OS buffer on COMMIT, fsync every second | 1-second data loss risk on OS crash | Medium |
| `0` | Write and fsync every second | 1-second data loss risk on MySQL crash | Highest write TPS |

```ini
[mysqld]
innodb_log_buffer_size          = 134217728  # 128 MB
innodb_flush_log_at_trx_commit  = 1          # default: full ACID
```

For non-critical bulk imports:

```sql
-- Temporarily reduce durability for a bulk load session
SET SESSION innodb_flush_log_at_trx_commit = 2;

LOAD DATA INFILE '/tmp/import.csv'
INTO TABLE orders
FIELDS TERMINATED BY ','
LINES TERMINATED BY '\n';

-- Restore default after load
SET SESSION innodb_flush_log_at_trx_commit = 1;
```

## Large transaction optimization

When inserting millions of rows in a single transaction the log buffer fills rapidly. Increasing it prevents constant mid-transaction flushes:

```sql
-- Check log buffer usage during a large import
SHOW ENGINE INNODB STATUS\G
-- Look for: LOG section
/*
---
LOG
---
Log sequence number          9876543210
Log buffer assigned up to    9876543210
Log buffer completed up to   9876543210
Log written up to            9876543210
...
*/

-- If 'Log buffer assigned' frequently equals 'Log written', buffer is keeping up
-- If there are gaps or 'log_waits' increases, enlarge the buffer
```

## Monitoring log buffer efficiency

```sql
-- Monitor key log metrics over time
SELECT
  variable_name,
  variable_value
FROM performance_schema.global_status
WHERE variable_name IN (
  'Innodb_log_writes',
  'Innodb_log_write_requests',
  'Innodb_log_waits',
  'Innodb_os_log_fsyncs',
  'Innodb_os_log_written'
);
```

## Log buffer vs redo log files

It is important not to confuse the log buffer with the redo log files:

| | Log buffer | Redo log files |
|---|---|---|
| Location | RAM | Disk |
| Variable | `innodb_log_buffer_size` | `innodb_redo_log_capacity` (8.0.30+) |
| Purpose | Staging area before disk write | Persistent crash recovery record |
| Typical size | 16 MB - 512 MB | 100 MB - 16 GB |

## Sizing formula

A rough guideline:

```text
innodb_log_buffer_size >= 2 * (average transaction size in bytes)
```

For a workload with many small OLTP transactions the default 16 MB is fine. For workloads generating large transactions (bulk inserts, big UPDATEs) set it to 128 MB or higher.

## Summary

The InnoDB log buffer is an in-memory staging area for redo log entries before they are flushed to disk. Its size is controlled by `innodb_log_buffer_size` (default 16 MB), which is dynamically adjustable in MySQL 8.0. Increase it to 64 MB - 512 MB when `Innodb_log_waits` is non-zero, when running large transactions, or during bulk data loads. A larger log buffer reduces the frequency of mid-transaction flushes and improves write throughput, at the cost of slightly more memory. Pair the log buffer configuration with an appropriate `innodb_flush_log_at_trx_commit` setting to balance durability and performance.
