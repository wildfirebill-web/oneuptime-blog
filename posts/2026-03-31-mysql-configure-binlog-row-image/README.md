# How to Configure binlog_row_image in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Binary Log, Replication, Configuration, Performance

Description: Learn how to configure binlog_row_image in MySQL to control what row data is written to the binary log in ROW format.

---

When using ROW format binary logging, MySQL records the actual row data for every change. The `binlog_row_image` variable controls how much of each row is written - the before image (old values), the after image (new values), or both. This setting has significant implications for binary log size, replication accuracy, and CDC (Change Data Capture) tools.

## What binlog_row_image Controls

For each row change in ROW format, MySQL can write:
- **Before image**: the row data before the change (relevant for UPDATE and DELETE)
- **After image**: the row data after the change (relevant for UPDATE and INSERT)

`binlog_row_image` has three values:

- `FULL` (default): write all columns in both before and after images
- `MINIMAL`: write only the columns needed to identify the row (primary key or unique index) plus changed columns
- `NOBLOB`: write all columns except unchanged BLOB/TEXT columns

## Checking the Current Setting

```sql
SHOW VARIABLES LIKE 'binlog_row_image';
```

## Setting binlog_row_image

Dynamically:

```sql
SET GLOBAL binlog_row_image = 'MINIMAL';
```

In `my.cnf` for persistence:

```ini
[mysqld]
binlog_format = ROW
binlog_row_image = MINIMAL
```

## FULL Mode - Maximum Safety

```ini
[mysqld]
binlog_row_image = FULL
```

With FULL, every before and after image contains all columns. This is the safest option because replicas and CDC tools can always reconstruct the complete row state without needing to read from the source.

Example binary log entry (conceptual) for an UPDATE:

```text
### UPDATE `mydb`.`orders`
### WHERE
###   @1=42        (id)
###   @2='pending' (status)
###   @3=99.99     (amount)
###   @4=1234      (customer_id)
### SET
###   @1=42
###   @2='shipped'
###   @3=99.99
###   @4=1234
```

## MINIMAL Mode - Reduced Log Size

```ini
[mysqld]
binlog_row_image = MINIMAL
```

MINIMAL only writes the primary key (for the WHERE clause) and changed columns (for the SET clause):

```text
### UPDATE `mydb`.`orders`
### WHERE
###   @1=42        (id - primary key only)
### SET
###   @2='shipped' (only changed column)
```

This can dramatically reduce binary log size - in some cases by 80-90% for tables with many columns where only a few change per update.

## NOBLOB Mode - Balance for Large Objects

```ini
[mysqld]
binlog_row_image = NOBLOB
```

NOBLOB behaves like FULL but excludes unchanged BLOB, TEXT, JSON, and GEOMETRY columns. This is useful for tables containing large text or binary data that rarely changes:

```sql
CREATE TABLE articles (
  id INT PRIMARY KEY,
  title VARCHAR(255),
  content LONGTEXT,   -- excluded from before image if unchanged
  updated_at DATETIME
);
```

## Impact on CDC and Replication Tools

Some CDC tools (Debezium, Maxwell, Tungsten Replicator) require FULL mode to capture complete before/after row state:

```ini
[mysqld]
# Required for Debezium and most CDC tools
binlog_row_image = FULL
binlog_row_metadata = FULL  # Also needed for column names in some tools
```

Standard MySQL replication works correctly with all three modes, as the replica uses the primary key for row lookup.

## Monitoring Binary Log Size Impact

Compare binary log generation with different settings:

```sql
-- Check current binary log position
SHOW MASTER STATUS;

-- Run a batch update
UPDATE orders SET status = 'processed' WHERE status = 'pending' LIMIT 1000;

-- Check new position to calculate bytes written
SHOW MASTER STATUS;
```

The difference in position bytes shows how much data was written to the binary log.

## Summary

`binlog_row_image = MINIMAL` significantly reduces binary log size and is appropriate for standard replication where tables have good primary keys. Use `FULL` when CDC tools like Debezium require complete before/after images or when troubleshooting replication issues. `NOBLOB` is a practical middle ground for schemas with large BLOB/TEXT columns that change infrequently. Set the value in `my.cnf` and restart the server, or apply it dynamically with `SET GLOBAL`.
