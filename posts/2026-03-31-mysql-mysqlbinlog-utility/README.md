# How to Use mysqlbinlog Utility in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Binary Log, mysqlbinlog, Recovery, Replication

Description: Learn how to use the mysqlbinlog utility to read, filter, and replay MySQL binary log files for point-in-time recovery and replication troubleshooting.

---

`mysqlbinlog` is the official MySQL utility for reading binary log files in human-readable format and replaying them against a MySQL server. It is essential for point-in-time recovery, replication troubleshooting, and auditing data changes. Binary logs record all data-modifying statements in the order they were executed.

## Basic Usage - Reading a Binary Log

```bash
mysqlbinlog /var/lib/mysql/mysql-bin.000001
```

This outputs the contents of the binary log file as SQL statements.

## Listing Available Binary Logs

```sql
SHOW BINARY LOGS;
```

```text
+------------------+-----------+
| Log_name         | File_size |
+------------------+-----------+
| mysql-bin.000001 |   1048576 |
| mysql-bin.000002 |    524288 |
| mysql-bin.000003 |      1024 |
+------------------+-----------+
```

## Filtering by Time Range

```bash
# Show events in a specific time window
mysqlbinlog --no-defaults \
  --start-datetime="2026-03-31 08:00:00" \
  --stop-datetime="2026-03-31 10:00:00" \
  /var/lib/mysql/mysql-bin.000001
```

## Filtering by Position

```bash
# Show events from a specific byte position
mysqlbinlog --no-defaults \
  --start-position=4 \
  --stop-position=500000 \
  /var/lib/mysql/mysql-bin.000001
```

## Reading Multiple Binary Log Files

```bash
mysqlbinlog --no-defaults \
  /var/lib/mysql/mysql-bin.000001 \
  /var/lib/mysql/mysql-bin.000002 \
  /var/lib/mysql/mysql-bin.000003
```

## Reading from a Remote Server

```bash
mysqlbinlog --no-defaults \
  --read-from-remote-server \
  --host=db.example.com \
  --user=replication_user \
  --password \
  mysql-bin.000001
```

## Decoding ROW Format Events

Binary logs in ROW format show base64-encoded row images by default. Use `--base64-output=DECODE-ROWS --verbose` to see readable SQL:

```bash
mysqlbinlog --no-defaults \
  --base64-output=DECODE-ROWS \
  --verbose \
  /var/lib/mysql/mysql-bin.000001
```

This converts row events into annotated comments showing before and after values:

```sql
### UPDATE `myapp`.`orders`
### WHERE
###   @1=1234 /* INT meta=0 nullable=0 */
###   @4='pending'
### SET
###   @4='shipped'
```

## Filtering by Database

```bash
mysqlbinlog --no-defaults \
  --database=myapp \
  /var/lib/mysql/mysql-bin.000001
```

## Replaying Binary Logs Against MySQL

```bash
# Pipe directly into mysql client
mysqlbinlog --no-defaults \
  --start-datetime="2026-03-31 02:00:00" \
  --stop-datetime="2026-03-31 09:44:59" \
  /var/lib/mysql/mysql-bin.000001 \
  | mysql -u root -p

# Or save to file first
mysqlbinlog --no-defaults \
  /var/lib/mysql/mysql-bin.000001 \
  > /tmp/replay.sql

mysql -u root -p < /tmp/replay.sql
```

## Finding a Specific Table's Changes

```bash
mysqlbinlog --no-defaults \
  --base64-output=DECODE-ROWS \
  --verbose \
  /var/lib/mysql/mysql-bin.000001 \
  | grep -A 5 "orders"
```

## Checking Binary Log Format

```sql
SHOW VARIABLES LIKE 'binlog_format';
```

- `ROW` - Records actual row changes (most complete, recommended)
- `STATEMENT` - Records SQL statements (compact but less reliable for some queries)
- `MIXED` - Uses STATEMENT normally, switches to ROW for non-deterministic queries

## Summary

`mysqlbinlog` is indispensable for point-in-time recovery and change auditing. Use `--start-datetime`/`--stop-datetime` or `--start-position`/`--stop-position` to filter relevant events. Add `--base64-output=DECODE-ROWS --verbose` to decode row format events into readable SQL. Always test binary log recovery procedures in a staging environment before relying on them in a production incident.
