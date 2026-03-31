# How to Use SHOW BINLOG EVENTS in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Binary Logs, SHOW BINLOG EVENTS, Replication, Auditing

Description: Learn how to use SHOW BINLOG EVENTS in MySQL to inspect binary log contents, trace specific transactions, and support point-in-time recovery.

---

## What Is SHOW BINLOG EVENTS

`SHOW BINLOG EVENTS` reads the contents of a binary log file and displays the events it contains. Each event represents a data modification (INSERT, UPDATE, DELETE) or structural change (DDL) that MySQL recorded. This command is useful for auditing changes, diagnosing replication issues, and identifying specific events to replay for point-in-time recovery.

## Basic Syntax

```sql
SHOW BINLOG EVENTS
    [IN 'log_name']
    [FROM pos]
    [LIMIT [offset,] row_count];
```

Without `IN`, MySQL reads the first binary log file.

## Viewing the Current Binary Log

First, check which log files exist:

```sql
SHOW BINARY LOGS;
```

Then inspect the current log:

```sql
SHOW BINLOG EVENTS IN 'mysql-bin.000003'\G
```

## Example Output

```text
*************************** 1. row ***************************
   Log_name: mysql-bin.000003
        Pos: 4
 Event_type: Format_desc
  Server_id: 1
End_log_pos: 125
       Info: Server ver: 8.0.32, Binlog ver: 4
*************************** 2. row ***************************
   Log_name: mysql-bin.000003
        Pos: 125
 Event_type: Previous_gtids
  Server_id: 1
End_log_pos: 196
       Info: ...
*************************** 3. row ***************************
   Log_name: mysql-bin.000003
        Pos: 196
 Event_type: Query
  Server_id: 1
End_log_pos: 295
       Info: BEGIN
```

Column meanings:

- `Pos` - byte offset where this event starts
- `Event_type` - type of event (Query, Write_rows, Update_rows, etc.)
- `End_log_pos` - byte offset where this event ends
- `Info` - event-specific detail

## Reading from a Specific Position

```sql
SHOW BINLOG EVENTS IN 'mysql-bin.000003' FROM 196 LIMIT 10;
```

## Limiting Output

```sql
-- Show first 20 events
SHOW BINLOG EVENTS IN 'mysql-bin.000003' LIMIT 20;

-- Show 20 events starting from offset 100
SHOW BINLOG EVENTS IN 'mysql-bin.000003' LIMIT 100, 20;
```

## Common Event Types

| Event_type | Meaning |
|---|---|
| Format_desc | Binary log file header |
| Query | DDL statement or BEGIN |
| Table_map | Identifies the table for row events |
| Write_rows | INSERT in ROW format |
| Update_rows | UPDATE in ROW format |
| Delete_rows | DELETE in ROW format |
| Xid | COMMIT for a transaction |
| Rotate | Switch to next binary log file |
| Gtid | GTID for a transaction |

## Reading Binary Logs with mysqlbinlog

For detailed row-level event content, `mysqlbinlog` is more powerful:

```bash
mysqlbinlog --base64-output=DECODE-ROWS --verbose \
    /var/lib/mysql/mysql-bin.000003 | head -100
```

Filter by time range:

```bash
mysqlbinlog \
    --start-datetime="2026-03-31 14:00:00" \
    --stop-datetime="2026-03-31 14:30:00" \
    /var/lib/mysql/mysql-bin.000003
```

Filter by database:

```bash
mysqlbinlog --database=mydb /var/lib/mysql/mysql-bin.000003
```

## Point-in-Time Recovery with Binary Logs

After restoring a full backup, replay binary logs to a specific point:

```bash
# Restore full backup
mysql -u root -p mydb < full_backup.sql

# Replay binary logs from backup position to a target time
mysqlbinlog \
    --start-position=4 \
    --stop-datetime="2026-03-31 13:59:59" \
    /var/lib/mysql/mysql-bin.000003 \
    /var/lib/mysql/mysql-bin.000004 \
    | mysql -u root -p mydb
```

## Finding Transactions by GTID

With GTID-based replication:

```sql
SHOW BINLOG EVENTS IN 'mysql-bin.000003'\G
-- Look for Gtid events, then find the corresponding transaction
```

```bash
mysqlbinlog --include-gtids="3E11FA47-71CA-11E1-9E33-C80AA9429562:23" \
    /var/lib/mysql/mysql-bin.000003
```

## Summary

`SHOW BINLOG EVENTS` provides a readable view into MySQL binary log contents at the event level, useful for auditing, replication troubleshooting, and identifying positions for point-in-time recovery. For detailed row data in ROW format logs, combine it with `mysqlbinlog --base64-output=DECODE-ROWS --verbose` for human-readable output.
