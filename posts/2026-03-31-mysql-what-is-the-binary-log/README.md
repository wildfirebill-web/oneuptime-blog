# What Is the Binary Log in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Binary Log, Replication, Point-in-Time Recovery, Change Data Capture

Description: The MySQL binary log records every change made to the database, enabling replication, point-in-time recovery, and change data capture.

---

## Overview

The binary log (binlog) is a sequential record of all events that change the MySQL database -- data modifications (`INSERT`, `UPDATE`, `DELETE`) and DDL statements (`CREATE TABLE`, `DROP TABLE`). Each binlog event includes the original SQL statement or row images (depending on format) and a timestamp. The binary log is the foundation of MySQL replication (replicas read it to apply changes) and point-in-time recovery (restoring from a backup and replaying events up to a specific time).

## Enabling the Binary Log

The binary log is enabled by default in MySQL 8.0. For MySQL 5.7, add to `my.cnf`:

```ini
[mysqld]
log_bin = /var/log/mysql/mysql-bin
server_id = 1
binlog_expire_logs_seconds = 604800  # 7 days
max_binlog_size = 256M
```

Verify it is enabled:

```sql
SHOW VARIABLES LIKE 'log_bin';
-- Value: ON

SHOW BINARY LOGS;
-- Lists all current binary log files with their sizes
```

## Binary Log Formats

Three formats control how events are recorded:

**STATEMENT**: Records the SQL statement itself.
```ini
binlog_format = STATEMENT
```
Compact, but some non-deterministic statements (using `NOW()`, `UUID()`) may replicate incorrectly.

**ROW** (recommended): Records the before and after images of each changed row.
```ini
binlog_format = ROW
```
Larger logs, but deterministic and safe for all statement types.

**MIXED**: Uses STATEMENT for safe statements, automatically switches to ROW for non-deterministic ones.

## Viewing Binary Log Contents

Use `mysqlbinlog` to read binary log files:

```bash
# Read a specific file
mysqlbinlog /var/log/mysql/mysql-bin.000001

# Filter by time range
mysqlbinlog --start-datetime="2026-03-31 10:00:00" \
            --stop-datetime="2026-03-31 11:00:00" \
            /var/log/mysql/mysql-bin.000001

# Read directly from a running server
mysqlbinlog --read-from-remote-server --host=localhost \
            --user=root --password mysql-bin.000001
```

## Point-in-Time Recovery

After restoring from a full backup, replay the binary log to recover up to a specific point:

```bash
# Step 1: restore from full backup (e.g., mysqldump)
mysql -u root -p < full_backup.sql

# Step 2: replay binary log events from backup time to target time
mysqlbinlog --start-datetime="2026-03-30 23:00:00" \
            --stop-datetime="2026-03-31 14:30:00" \
            /var/log/mysql/mysql-bin.* | mysql -u root -p
```

## Managing Binary Log Retention

```sql
-- Manually purge logs older than a date
PURGE BINARY LOGS BEFORE '2026-03-24 00:00:00';

-- Purge up to a specific log file
PURGE BINARY LOGS TO 'mysql-bin.000010';

-- Set automatic expiration (seconds)
SET GLOBAL binlog_expire_logs_seconds = 604800;
```

## Viewing Events Inside the MySQL Client

```sql
-- List binary log files
SHOW BINARY LOGS;

-- View events in a specific log file
SHOW BINLOG EVENTS IN 'mysql-bin.000001' LIMIT 20;

-- View the current position
SHOW MASTER STATUS;
```

## Summary

The MySQL binary log is a sequential record of all data-changing events used for replication, point-in-time recovery, and change data capture. The ROW format is recommended for correctness and compatibility with tools. Binary logs must be retained long enough to cover your recovery point objective and replication lag. Understanding the binary log is foundational to MySQL operations, disaster recovery planning, and building event-driven data pipelines.
