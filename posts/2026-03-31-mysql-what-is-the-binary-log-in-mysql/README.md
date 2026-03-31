# What Is the Binary Log in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Binary Log, Replication, Point-In-Time Recovery

Description: The MySQL binary log records all changes to the database as a sequence of events, enabling replication and point-in-time recovery.

---

## Overview

The binary log (binlog) is a set of log files that contain a record of all data modification statements and data definition changes made to a MySQL server. Unlike the general query log, the binary log only records statements that change data - not SELECT queries.

The binary log serves two primary purposes:
- **Replication** - replicas read the primary's binary log to stay in sync
- **Point-in-time recovery** - apply binlog events after restoring a full backup

## Enabling the Binary Log

In MySQL 8.0, the binary log is enabled by default. To configure it in `my.cnf`:

```text
[mysqld]
log_bin = /var/log/mysql/mysql-bin
server_id = 1
binlog_format = ROW
binlog_expire_logs_seconds = 604800
```

## Viewing Binary Log Status

```sql
-- Check if binary logging is on
SHOW VARIABLES LIKE 'log_bin';

-- List all binary log files
SHOW BINARY LOGS;

-- Show the current binary log position
SHOW MASTER STATUS;
```

## Binary Log Formats

MySQL supports three binary log formats:

```sql
SHOW VARIABLES LIKE 'binlog_format';
```

| Format | Description |
|--------|-------------|
| `STATEMENT` | Logs the SQL statement (compact, but non-deterministic functions are risky) |
| `ROW` | Logs the actual row changes (safe, verbose) |
| `MIXED` | Uses STATEMENT by default, falls back to ROW when needed |

MySQL 8.0 defaults to `ROW` format, which is the safest for replication.

## Reading Binary Log Events

Use the `mysqlbinlog` utility to read events:

```bash
# Read a specific binary log file
mysqlbinlog /var/log/mysql/mysql-bin.000001

# Filter by time range
mysqlbinlog --start-datetime="2026-03-01 00:00:00" \
            --stop-datetime="2026-03-01 01:00:00" \
            /var/log/mysql/mysql-bin.000001

# Filter by position
mysqlbinlog --start-position=1234 --stop-position=5678 \
            /var/log/mysql/mysql-bin.000001
```

## Using SHOW BINLOG EVENTS

```sql
-- View events in the current binary log
SHOW BINLOG EVENTS;

-- View events in a specific file from a given position
SHOW BINLOG EVENTS IN 'mysql-bin.000001' FROM 4 LIMIT 10;
```

## Point-in-Time Recovery Example

```bash
# Step 1: Restore from full backup
mysql -u root -p < full_backup.sql

# Step 2: Apply binary log events from backup time to now
mysqlbinlog --start-datetime="2026-03-30 02:00:00" \
            /var/log/mysql/mysql-bin.000001 | mysql -u root -p
```

## Managing Binary Log Retention

```sql
-- Set expiry in seconds (7 days)
SET GLOBAL binlog_expire_logs_seconds = 604800;

-- Manually purge logs older than a date
PURGE BINARY LOGS BEFORE '2026-03-25 00:00:00';

-- Purge up to a specific log file
PURGE BINARY LOGS TO 'mysql-bin.000010';
```

## Checking Binary Log Size

```bash
ls -lh /var/log/mysql/mysql-bin.*
```

```sql
-- View log sizes from within MySQL
SHOW BINARY LOGS;
-- +------------------+-----------+
-- | Log_name         | File_size |
-- +------------------+-----------+
-- | mysql-bin.000001 | 524288    |
-- | mysql-bin.000002 |  1024     |
```

## Summary

The MySQL binary log is a critical feature for replication and disaster recovery. It records every data-changing event in the server and can be used to replay changes for point-in-time recovery. With ROW format enabled by default in MySQL 8.0, the binary log provides reliable, deterministic change tracking that underpins both replication pipelines and backup strategies.
