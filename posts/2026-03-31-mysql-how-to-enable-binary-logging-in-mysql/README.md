# How to Enable Binary Logging in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Mysql, Binary Log, Replication, Recovery, Configuration

Description: Learn how to enable and configure MySQL binary logging for replication, point-in-time recovery, and change data capture with practical configuration examples.

---

## What Is MySQL Binary Logging?

The MySQL binary log (binlog) is a sequential log file that records all DDL and DML statements that change data. It serves three main purposes:

1. **Replication** - source servers send binlog events to replicas
2. **Point-in-time recovery** - replay events from a backup point to restore to a specific time
3. **Change data capture** - tools like Debezium read the binlog to stream changes to other systems

## Enabling Binary Logging

Binary logging is enabled in `my.cnf`:

```text
[mysqld]
log_bin = /var/log/mysql/mysql-bin
server_id = 1
```

- `log_bin` - path prefix for binary log files. MySQL appends sequence numbers: `mysql-bin.000001`, `mysql-bin.000002`, etc.
- `server_id` - required for binary logging; must be unique across all MySQL instances in a replication topology.

Restart MySQL to apply:

```bash
sudo systemctl restart mysql
```

## Verifying Binary Logging Is Active

```sql
SHOW VARIABLES LIKE 'log_bin';
```

```text
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| log_bin       | ON    |
+---------------+-------+
```

```sql
SHOW BINARY LOGS;
```

```text
+------------------+-----------+
| Log_name         | File_size |
+------------------+-----------+
| mysql-bin.000001 | 524288000 |
| mysql-bin.000002 | 4096      |
+------------------+-----------+
```

## Choosing the Binary Log Format

MySQL supports three binlog formats:

| Format | Description | Best For |
|---|---|---|
| `ROW` | Logs the before/after image of each changed row | Replication accuracy, CDC |
| `STATEMENT` | Logs the SQL statement | Smaller log files |
| `MIXED` | Uses STATEMENT by default, ROW for non-deterministic statements | Balanced |

```text
[mysqld]
binlog_format = ROW
```

`ROW` format is strongly recommended for replication reliability and PITR. Use `binlog_row_image = MINIMAL` to reduce row format log size:

```text
[mysqld]
binlog_format = ROW
binlog_row_image = MINIMAL
```

## Binary Log Retention

Control how long binary logs are kept:

```text
[mysqld]
# MySQL 8.0 (in seconds - 7 days)
binlog_expire_logs_seconds = 604800

# MySQL 5.7 (in days)
expire_logs_days = 7
```

Manually purge logs older than a specific date:

```sql
PURGE BINARY LOGS BEFORE '2024-05-01 00:00:00';
```

Purge up to a specific log file:

```sql
PURGE BINARY LOGS TO 'mysql-bin.000050';
```

## Viewing Binary Log Contents

Use `mysqlbinlog` to read binary log files:

```bash
mysqlbinlog /var/log/mysql/mysql-bin.000002
```

Filter by time range:

```bash
mysqlbinlog \
  --start-datetime="2024-05-10 14:00:00" \
  --stop-datetime="2024-05-10 15:00:00" \
  /var/log/mysql/mysql-bin.000002
```

## Binary Log Checksums (MySQL 5.6+)

Enable checksums to detect corruption in binary log events:

```text
[mysqld]
binlog_checksum = CRC32
```

## Binary Log Encryption (MySQL 8.0.14+)

Encrypt binary log files at rest:

```sql
SET GLOBAL binlog_encryption = ON;
```

Or in `my.cnf`:

```text
[mysqld]
binlog_encryption = ON
```

## Flushing Binary Logs

Force rotation to a new binary log file:

```sql
FLUSH BINARY LOGS;
```

Check current binary log position:

```sql
SHOW MASTER STATUS;
```

```text
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000002 | 4321     |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
```

## Summary

Enable MySQL binary logging by setting `log_bin` and `server_id` in `my.cnf`. Use `binlog_format = ROW` for reliable replication and point-in-time recovery. Configure retention with `binlog_expire_logs_seconds` to avoid filling up disk space. Binary logs are required for replication, PITR, and change data capture - enable them before they are needed.
