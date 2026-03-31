# How to Use the mysqlbinlog Tool to Read Binary Logs in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Binary Log, Recovery, Tool, Administration

Description: Learn how to use the mysqlbinlog command-line tool to read, filter, and replay MySQL binary log files for recovery and auditing.

---

`mysqlbinlog` is the command-line utility for reading and interpreting MySQL binary log files. It converts the binary format into human-readable SQL statements or replays events directly into a MySQL server. It is the primary tool for point-in-time recovery and binary log analysis.

## Basic Usage

Read a binary log file and print its contents:

```bash
mysqlbinlog /var/lib/mysql/binlogs/mysql-bin.000003
```

For ROW format logs, add `--verbose` to decode row events into pseudo-SQL:

```bash
mysqlbinlog --verbose /var/lib/mysql/binlogs/mysql-bin.000003
```

## Filtering by Date and Time

Extract events within a specific time window:

```bash
mysqlbinlog \
  --start-datetime="2024-03-15 14:00:00" \
  --stop-datetime="2024-03-15 14:30:00" \
  /var/lib/mysql/binlogs/mysql-bin.000003
```

## Filtering by Binary Log Position

Extract events between two positions (useful after `SHOW BINLOG EVENTS`):

```bash
mysqlbinlog \
  --start-position=157 \
  --stop-position=2450 \
  /var/lib/mysql/binlogs/mysql-bin.000003
```

## Filtering by Database

Only extract events for a specific database:

```bash
mysqlbinlog \
  --database=mydb \
  /var/lib/mysql/binlogs/mysql-bin.000003
```

Note: `--database` filters at the event level and may miss some cross-database operations.

## Reading Multiple Binary Log Files

For point-in-time recovery spanning multiple binary log files, pass them all at once:

```bash
mysqlbinlog \
  --start-datetime="2024-03-15 00:00:00" \
  --stop-datetime="2024-03-15 23:59:59" \
  /var/lib/mysql/binlogs/mysql-bin.000001 \
  /var/lib/mysql/binlogs/mysql-bin.000002 \
  /var/lib/mysql/binlogs/mysql-bin.000003 \
  | mysql -u root -p
```

## Point-in-Time Recovery Workflow

Typical recovery after accidental data loss:

```bash
# Step 1: Restore the last full backup
mysql -u root -p mydb < /var/backups/mysql/full_20240315.sql

# Step 2: Find the position of the accidental DELETE or DROP
mysqlbinlog --verbose /var/lib/mysql/binlogs/mysql-bin.000010 | grep -B 5 "DROP TABLE"

# Step 3: Apply all events up to just before the bad event
mysqlbinlog \
  --stop-position=8432 \
  /var/lib/mysql/binlogs/mysql-bin.000010 \
  | mysql -u root -p mydb

echo "Recovery complete"
```

## Reading Remote Binary Logs

Connect to a remote MySQL server and read its binary logs directly:

```bash
mysqlbinlog \
  --read-from-remote-server \
  --host=192.168.1.10 \
  --user=repl_user \
  --password=ReplPassword \
  --raw \
  --result-file=/tmp/remote-binlogs/ \
  mysql-bin.000003
```

## Decoding Row Events with Base64

ROW format events are base64-encoded. Use `--base64-output=DECODE-ROWS` with `--verbose` for readable output:

```bash
mysqlbinlog \
  --base64-output=DECODE-ROWS \
  --verbose \
  /var/lib/mysql/binlogs/mysql-bin.000003
```

## Saving Output to a File

Save decoded output for review before applying:

```bash
mysqlbinlog \
  --start-datetime="2024-03-15 14:00:00" \
  --stop-datetime="2024-03-15 15:00:00" \
  --verbose \
  /var/lib/mysql/binlogs/mysql-bin.000003 \
  > /tmp/recovery.sql

# Review the file before applying
less /tmp/recovery.sql

# Apply after verification
mysql -u root -p mydb < /tmp/recovery.sql
```

## Summary

`mysqlbinlog` is the essential tool for reading MySQL binary logs and performing point-in-time recovery. Use `--start-datetime` and `--stop-datetime` for time-based filtering, and `--start-position` / `--stop-position` for precise event selection. Always add `--verbose` for ROW format logs to decode row changes into readable SQL. For recovery, save output to a file, review it before applying, and replay events into MySQL with a piped `mysql` command.
