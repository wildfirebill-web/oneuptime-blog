# How to Enable Binary Logging in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Binary Log, Replication, Backup, Recovery, Configuration

Description: Learn how to enable and configure binary logging in MySQL to support point-in-time recovery, replication, and change data capture.

---

The MySQL binary log records all data-modifying statements and row changes. It is a prerequisite for replication, enables point-in-time recovery from backups, and supports change data capture (CDC) for event-driven architectures. This guide covers how to enable and configure it.

## Check Whether Binary Logging is Enabled

```sql
SHOW VARIABLES LIKE 'log_bin';
SHOW VARIABLES LIKE 'log_bin_basename';
```

If `log_bin` is `OFF`, binary logging is disabled. On MySQL 8.0, binary logging is enabled by default, but it may be disabled on some installations or containers.

## Enable Binary Logging

Edit the MySQL configuration file:

```bash
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

Add or update these settings under `[mysqld]`:

```text
[mysqld]
log_bin = /var/log/mysql/mysql-bin
server-id = 1
binlog_format = ROW
expire_logs_days = 7
max_binlog_size = 100M
```

Parameter explanations:

```text
log_bin           - Path prefix for binary log files; enables logging
server-id         - Unique ID for this server (required for replication)
binlog_format     - ROW (recommended), STATEMENT, or MIXED
expire_logs_days  - Automatically purge logs older than N days
max_binlog_size   - Rotate to a new file after this size is reached
```

Restart MySQL to apply the settings:

```bash
sudo systemctl restart mysql
```

## Verify Binary Logging is Active

After restarting, confirm binary logging is enabled:

```sql
SHOW VARIABLES LIKE 'log_bin';
SHOW BINARY LOGS;
```

The `SHOW BINARY LOGS` command lists all current binary log files with their sizes:

```text
+------------------+-----------+
| Log_name         | File_size |
+------------------+-----------+
| mysql-bin.000001 |       154 |
+------------------+-----------+
```

## Choose the Right Binary Log Format

MySQL supports three formats:

```text
ROW       - Records exact row changes; safe for all statement types (recommended)
STATEMENT - Records the SQL statement; smaller files but unsafe with non-deterministic functions
MIXED     - Uses STATEMENT by default, falls back to ROW for unsafe statements
```

ROW format is recommended for replication and CDC because it captures the exact data that changed. Set it explicitly:

```sql
SET GLOBAL binlog_format = 'ROW';
```

Or in `mysqld.cnf`:

```text
[mysqld]
binlog_format = ROW
binlog_row_image = FULL
```

`binlog_row_image = FULL` captures all columns in each row change, which is required by most CDC tools.

## View Binary Log Events

To inspect what is recorded in a binary log file from inside MySQL:

```sql
SHOW BINLOG EVENTS IN 'mysql-bin.000001' LIMIT 20;
```

From the command line using the `mysqlbinlog` tool:

```bash
mysqlbinlog /var/log/mysql/mysql-bin.000001
```

For ROW format, add the `--verbose` flag to display human-readable row changes:

```bash
mysqlbinlog --verbose /var/log/mysql/mysql-bin.000001
```

## Manage Binary Log Retention

Set automatic expiration using the `binlog_expire_logs_seconds` variable (MySQL 8.0):

```sql
SET GLOBAL binlog_expire_logs_seconds = 604800;  -- 7 days
```

Or in `mysqld.cnf`:

```text
[mysqld]
binlog_expire_logs_seconds = 604800
```

To manually purge old binary logs up to a specific file:

```sql
PURGE BINARY LOGS TO 'mysql-bin.000010';
```

To purge logs older than a specific date:

```sql
PURGE BINARY LOGS BEFORE '2026-03-01 00:00:00';
```

## Check Binary Log Disk Usage

```sql
SHOW BINARY LOGS;
```

Or from the shell:

```bash
du -sh /var/log/mysql/mysql-bin.*
```

Monitor disk space carefully - on busy servers, binary logs can grow quickly.

## Flush to a New Binary Log File

To force MySQL to start writing to a new binary log file:

```sql
FLUSH BINARY LOGS;
```

This is useful before taking a backup so the backup corresponds to a clean log boundary.

## Summary

Binary logging in MySQL is enabled by adding `log_bin` and `server-id` to `mysqld.cnf` and restarting the server. Use `binlog_format = ROW` for safe replication and CDC. Verify the configuration with `SHOW VARIABLES LIKE 'log_bin'` and manage retention with `binlog_expire_logs_seconds`. Once enabled, the binary log serves as the foundation for replication, point-in-time recovery, and change data capture.
