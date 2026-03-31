# How to Configure InnoDB Redo Log in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, InnoDB, Redo Log, Configuration, Durability

Description: Learn how to configure InnoDB redo log size and related settings in MySQL to balance write performance with crash recovery durability.

---

## What Is the InnoDB Redo Log?

The InnoDB redo log is a disk-based structure used for crash recovery. When data is modified, InnoDB writes the changes to the redo log before applying them to the actual data files. If MySQL crashes, InnoDB replays the redo log on restart to recover committed transactions.

Redo log size directly affects write throughput - a larger redo log allows more writes to be buffered before flushing, improving performance for write-heavy workloads.

## Redo Log Configuration in MySQL 5.7 and Earlier

In MySQL 5.7, redo log size is controlled by two variables:

```text
[mysqld]
innodb_log_file_size = 512M
innodb_log_files_in_group = 2
```

Total redo log capacity = `innodb_log_file_size` x `innodb_log_files_in_group`. In this example, the total is 1GB.

## Redo Log Configuration in MySQL 8.0+

MySQL 8.0.30 introduced a single variable to control total redo log capacity:

```sql
SET GLOBAL innodb_redo_log_capacity = 2147483648;  -- 2GB
```

Or in `my.cnf`:

```text
[mysqld]
innodb_redo_log_capacity = 2G
```

The redo log is now stored in individual files inside the `#innodb_redo` directory under the data directory.

## Checking Current Redo Log Configuration

```sql
-- MySQL 8.0.30+
SHOW VARIABLES LIKE 'innodb_redo_log_capacity';

-- MySQL 5.7 / earlier 8.0
SHOW VARIABLES LIKE 'innodb_log_file%';
```

## Choosing the Right Redo Log Size

A common sizing method uses the 1-hour rule: set the redo log large enough to hold 1 hour of write activity. You can estimate this from the LSN (Log Sequence Number) advancement:

```sql
-- Check current LSN
SHOW ENGINE INNODB STATUS\G
-- Look for: Log sequence number XXXXXXXX
-- Wait 60 minutes, run again, compute the difference
```

Alternatively, check writes per second using status variables:

```sql
SHOW GLOBAL STATUS LIKE 'Innodb_os_log_written';
-- Wait 60 seconds, run again
-- Bytes written per second = (new - old) / 60
```

## Impact of innodb_flush_log_at_trx_commit

The `innodb_flush_log_at_trx_commit` variable controls when redo log writes are flushed to disk:

| Value | Behavior | Durability | Performance |
|---|---|---|---|
| `1` (default) | Flush to disk on every commit | Full ACID | Lowest write throughput |
| `2` | Write to OS buffer on commit, flush every second | Lose up to 1 second | Moderate |
| `0` | Write to InnoDB buffer, flush every second | Lose up to 1 second | Highest |

```text
[mysqld]
innodb_flush_log_at_trx_commit = 1
```

For most production systems, keep this at `1`. Use `2` for read-heavy replicas or batch import jobs.

## Monitoring Redo Log Usage

```sql
-- MySQL 8.0.30+
SELECT * FROM performance_schema.innodb_redo_log_files;

-- Check redo log usage percentage
SELECT
  capacity,
  used,
  ROUND(used / capacity * 100, 2) AS used_pct
FROM (
  SELECT
    (SELECT VARIABLE_VALUE FROM performance_schema.global_status WHERE VARIABLE_NAME = 'Innodb_redo_log_capacity_resized') AS capacity,
    (SELECT VARIABLE_VALUE FROM performance_schema.global_status WHERE VARIABLE_NAME = 'Innodb_redo_log_logical_size') AS used
) AS rl;
```

If redo log usage stays consistently above 75%, increase the capacity.

## Summary

The InnoDB redo log ensures crash recovery by logging all changes before applying them. In MySQL 8.0.30+, use `innodb_redo_log_capacity` to set the total size. Size the redo log to hold at least 1 hour of write activity, and keep `innodb_flush_log_at_trx_commit = 1` in production for full ACID compliance.
