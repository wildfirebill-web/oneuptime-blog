# How MySQL Binary Logging Works Internally

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Binary Log, Replication, Recovery, Internals

Description: Understand how MySQL binary logging records data changes, supports replication and point-in-time recovery, and how to manage binary log files.

---

## What Is the Binary Log

The MySQL binary log (binlog) is a sequential log of all statements or row changes that modify data. It is used for:
- **Replication** - replicas read the binlog to replay changes.
- **Point-in-time recovery (PITR)** - restore to any point after a backup.
- **Auditing** - track all data changes with timestamps.

The binary log is not the InnoDB redo log (which handles crash recovery). They serve different purposes.

## Binary Log Formats

MySQL supports three formats, controlled by `binlog_format`:

| Format | What It Records | Pros | Cons |
|--------|----------------|------|------|
| `STATEMENT` | SQL statements | Compact, human-readable | Non-deterministic functions can cause replica divergence |
| `ROW` | Row images (before/after) | Exact and deterministic | Larger log files |
| `MIXED` | Statements by default, rows for unsafe statements | Balance of both | More complex |

```sql
SHOW VARIABLES LIKE 'binlog_format';
SET GLOBAL binlog_format = 'ROW';
```

The default in MySQL 8 is `ROW`.

## How Writes Reach the Binary Log

When a transaction commits (InnoDB):
1. InnoDB writes changes to the redo log (persists to disk via `fsync`).
2. The binlog is written and fsynced.
3. InnoDB marks the transaction as committed.

This two-phase commit (2PC) keeps the redo log and binary log synchronized.

The `sync_binlog` variable controls when the binlog is fsynced:
- `sync_binlog = 0` - OS decides when to flush (fast, but data loss risk).
- `sync_binlog = 1` - fsync after every transaction commit (safest, default in MySQL 8).
- `sync_binlog = N` - fsync every N commits (performance/durability tradeoff).

```sql
SHOW VARIABLES LIKE 'sync_binlog';
```

## Binary Log File Structure

MySQL writes binlog events to numbered files:

```text
mysql-bin.000001
mysql-bin.000002
...
mysql-bin.index  (index of all active log files)
```

Events are written sequentially. When a file reaches `max_binlog_size` (default 1GB), MySQL rotates to a new file. A rotation also happens on server restart or when `FLUSH BINARY LOGS` is executed.

```sql
SHOW BINARY LOGS;
-- Shows all current binary log files and their sizes
```

## Reading Binary Log Contents

Use `mysqlbinlog`:

```bash
mysqlbinlog /var/lib/mysql/mysql-bin.000001
```

Filter by time range:

```bash
mysqlbinlog --start-datetime="2026-03-31 10:00:00" \
            --stop-datetime="2026-03-31 11:00:00" \
            /var/lib/mysql/mysql-bin.000001
```

Filter by database:

```bash
mysqlbinlog --database=myapp /var/lib/mysql/mysql-bin.000001
```

## Row Image Control in ROW Format

With `binlog_format = ROW`, MySQL can log full or minimal row images:

```sql
SHOW VARIABLES LIKE 'binlog_row_image';
```

- `FULL` - logs all columns in before and after images (default).
- `MINIMAL` - logs only changed columns (smaller binlog, but less info for recovery).
- `NOBLOB` - like FULL but excludes unchanged BLOB/TEXT columns.

## Binary Log and GTID

GTID (Global Transaction Identifier) mode assigns a unique ID to every transaction:

```text
source_uuid:transaction_sequence_number
e.g., 3E11FA47-71CA-11E1-9E33-C80AA9429562:23
```

```sql
SHOW VARIABLES LIKE 'gtid_mode';
SHOW VARIABLES LIKE 'enforce_gtid_consistency';
```

GTIDs make it trivial to identify which transactions have been applied on each replica and to skip or re-apply specific transactions.

## Expiring Binary Logs

```sql
SHOW VARIABLES LIKE 'binlog_expire_logs_seconds';
-- Default: 2592000 (30 days in MySQL 8)
```

Set a shorter expiry for disk space:

```sql
SET GLOBAL binlog_expire_logs_seconds = 604800;  -- 7 days
```

Manually purge old logs:

```sql
PURGE BINARY LOGS TO 'mysql-bin.000050';
-- Or by date:
PURGE BINARY LOGS BEFORE '2026-03-24 00:00:00';
```

Never purge logs that have not yet been consumed by a replica - check `SHOW REPLICA STATUS` first.

## Point-in-Time Recovery with Binary Logs

After restoring a backup, replay binlog events to recover to a specific point:

```bash
# Restore full backup first
mysqlbackup --apply-log --backup-dir=/backup/2026-03-30 restore

# Then replay binlog events from the backup point to 10:30
mysqlbinlog --start-position=4 \
            --stop-datetime="2026-03-31 10:30:00" \
            mysql-bin.000060 mysql-bin.000061 | mysql -u root -p
```

## Summary

MySQL binary logging records every data-modifying transaction and is the foundation of replication and point-in-time recovery. ROW format is the safest for replication accuracy; sync_binlog=1 ensures durability. Use GTIDs for simplified replica management, and set appropriate expiry policies to balance disk space with recovery window. Always check replica consumption before purging logs.
