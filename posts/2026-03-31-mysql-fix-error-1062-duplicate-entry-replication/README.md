# How to Fix ERROR 1062 Duplicate Entry in MySQL Replication

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Replication, Error, Duplicate, Primary Key

Description: Fix MySQL replication ERROR 1062 Duplicate entry by skipping the conflicting event, deleting the duplicate row on the replica, or using REPLACE to handle conflicts.

---

MySQL replication ERROR 1062 occurs when the replica's SQL thread tries to insert a row that already exists. The `SHOW REPLICA STATUS` output shows: `Last_SQL_Error: Could not execute Write_rows event on table mydb.users; Duplicate entry '42' for key 'PRIMARY'`.

## Why This Happens

- A row was manually inserted directly on the replica
- The same insert was applied twice (e.g., from a previous skip of a DELETE event)
- A previous replication error caused the replica to be out of sync
- The replica had `read_only = OFF` and received a write from an application

## Diagnose the Error

```sql
-- On the replica
SHOW REPLICA STATUS\G
-- Note: Last_SQL_Errno: 1062
-- Note: Last_SQL_Error for table and key name
-- Note: Exec_Source_Log_File and Exec_Source_Log_Pos
```

Check the duplicate row on the replica:

```sql
-- Check what is on the replica
SELECT * FROM users WHERE id = 42;

-- Check the source for the same row
-- (connect to source and run)
SELECT * FROM users WHERE id = 42;
```

## Fix 1: Delete the Duplicate from the Replica

If the row on the replica is stale or incorrect:

```sql
-- On the replica, stop SQL thread
STOP REPLICA SQL_THREAD;

-- Delete the conflicting row
DELETE FROM users WHERE id = 42;

-- Resume replication
START REPLICA SQL_THREAD;

-- Verify
SHOW REPLICA STATUS\G
```

## Fix 2: Skip the Single Event

If you want to ignore the conflicting INSERT and keep the replica's existing row:

```sql
STOP REPLICA SQL_THREAD;

-- Skip one event
SET GLOBAL SQL_REPLICA_SKIP_COUNTER = 1;

START REPLICA SQL_THREAD;
SHOW REPLICA STATUS\G
```

## Fix 3: Configure slave_exec_mode to IDEMPOTENT

MySQL's `slave_exec_mode = IDEMPOTENT` makes the SQL thread handle 1062 and 1032 errors automatically by using REPLACE instead of INSERT and ignoring missing rows for DELETE:

```sql
SET GLOBAL slave_exec_mode = 'IDEMPOTENT';
START REPLICA SQL_THREAD;
```

Or permanently in `my.cnf` (use carefully - it masks data inconsistencies):

```text
[mysqld]
slave_exec_mode = IDEMPOTENT
```

## Fix 4: GTID Replication Skip

For GTID-based replication, skip the specific transaction:

```sql
STOP REPLICA;

-- Get the failing GTID from Executing_Gtid_Set or the error message
SET GTID_NEXT = 'source_uuid:failing_transaction_id';
BEGIN;
COMMIT;
SET GTID_NEXT = 'AUTOMATIC';

START REPLICA;
SHOW REPLICA STATUS\G
```

## Fix 5: Resync the Affected Table

If errors are recurring across many rows, resync the whole table:

```bash
# Dump from source
mysqldump -u root -p --single-transaction \
  --no-create-info mydb users > users_sync.sql
```

```sql
-- On the replica
STOP REPLICA SQL_THREAD;
TRUNCATE TABLE users;
```

```bash
mysql -u root -p mydb < users_sync.sql
```

```sql
START REPLICA SQL_THREAD;
```

## Prevent Future Duplicate Entries

Enable `read_only` on the replica to prevent direct writes:

```sql
SET GLOBAL read_only = ON;
SET GLOBAL super_read_only = ON;
```

## Summary

ERROR 1062 in replication means the replica already has a row that the source is trying to insert. The cleanest fix is to delete the duplicate from the replica and resume. For recurring errors, use `pt-table-sync` to synchronize data and enable `super_read_only` on all replicas to prevent accidental direct writes that cause future divergence.
