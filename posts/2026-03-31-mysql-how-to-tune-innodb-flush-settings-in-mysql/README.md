# How to Tune InnoDB Flush Settings in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, InnoDB, Performance Tuning, Flush Settings, Durability

Description: Learn how to tune InnoDB flush settings in MySQL including innodb_flush_log_at_trx_commit and innodb_flush_method to balance performance and durability.

---

## Overview of InnoDB Flush Settings

InnoDB uses two critical flush settings that control the trade-off between durability and write performance:

1. **`innodb_flush_log_at_trx_commit`** - controls when redo log buffers are flushed to disk on commit
2. **`innodb_flush_method`** - controls how InnoDB interacts with the OS file system to write data and log files

Getting these settings wrong either wastes performance (being too conservative) or risks data loss (being too aggressive).

## innodb_flush_log_at_trx_commit

This setting has three values:

| Value | Behavior | Durability | Performance |
|-------|----------|------------|-------------|
| 1 (default) | Flush and sync on every commit | Full ACID | Slowest |
| 2 | Write to OS cache on commit, sync every second | Lose up to 1 sec of data on OS crash | Fast |
| 0 | Flush every second, sync every second | Lose up to 1 sec of data on MySQL crash | Fastest |

For financial or critical data:

```text
[mysqld]
innodb_flush_log_at_trx_commit = 1
```

For analytics or staging where 1 second of data loss is acceptable:

```text
[mysqld]
innodb_flush_log_at_trx_commit = 2
```

Check the current setting:

```sql
SHOW VARIABLES LIKE 'innodb_flush_log_at_trx_commit';
```

## innodb_flush_method

The flush method controls how InnoDB calls the OS to persist data. Options depend on OS and storage:

| Value | Description | Best For |
|-------|-------------|----------|
| `fsync` | Default, uses fsync() | Most environments |
| `O_DIRECT` | Bypasses OS cache for data files | Dedicated servers with battery-backed RAID |
| `O_DIRECT_NO_FSYNC` | Like O_DIRECT but skips fsync | Modern SSDs with safe caches |
| `O_DSYNC` | Uses O_SYNC for log files | Some older Linux systems |

For a dedicated MySQL server with reliable storage (e.g., SSD, NVMe, or battery-backed RAID):

```text
[mysqld]
innodb_flush_method = O_DIRECT
```

`O_DIRECT` avoids double-buffering (data stored in both InnoDB buffer pool and OS page cache), which frees up OS memory and improves buffer pool efficiency.

For cloud environments (RDS, Cloud SQL, Aurora), leave `innodb_flush_method` at the default since the storage layer handles durability independently.

## innodb_flush_neighbors

On spinning disk (HDD), flushing neighboring dirty pages together reduces seek time. On SSDs, this offers no benefit and adds latency:

```text
# For SSD/NVMe storage (default in MySQL 8.0)
innodb_flush_neighbors = 0

# For HDD storage
innodb_flush_neighbors = 1
```

## innodb_log_buffer_size

The log buffer holds redo log data before it is flushed to the log file on disk. A larger buffer reduces disk writes for workloads with large transactions:

```text
[mysqld]
innodb_log_buffer_size = 64M
```

Check current usage:

```sql
SHOW STATUS LIKE 'Innodb_log_waits';
```

If `Innodb_log_waits` is greater than 0, the log buffer is too small.

## Recommended Settings for Different Workloads

For OLTP with strict durability:

```text
[mysqld]
innodb_flush_log_at_trx_commit = 1
innodb_flush_method = O_DIRECT
innodb_flush_neighbors = 0
innodb_log_buffer_size = 64M
```

For batch-heavy or analytics workloads:

```text
[mysqld]
innodb_flush_log_at_trx_commit = 2
innodb_flush_method = O_DIRECT
innodb_flush_neighbors = 0
innodb_log_buffer_size = 128M
```

## Verify Changes

```sql
SHOW VARIABLES LIKE 'innodb_flush%';
SHOW STATUS LIKE 'Innodb_log_waits';
SHOW STATUS LIKE 'Innodb_pages_written';
```

## Summary

InnoDB flush settings determine the balance between write durability and performance. For production OLTP, use `innodb_flush_log_at_trx_commit=1` for full ACID compliance and `innodb_flush_method=O_DIRECT` to avoid double-buffering on dedicated servers. For less critical or write-heavy workloads, `innodb_flush_log_at_trx_commit=2` provides a meaningful performance improvement with minimal risk.
