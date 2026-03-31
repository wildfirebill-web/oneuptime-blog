# How to Skip a Replication Error in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Replication, Troubleshooting, Database Administration

Description: Learn how to safely skip a replication error in MySQL using GTID mode or the SQL_SKIP_COUNTER variable to resume a stopped replica.

---

## When to Skip a Replication Error

Replication errors stop the SQL thread, halting replication until the error is resolved. Common causes include:

- Duplicate key violations (row already exists on the replica)
- Row not found errors (row was deleted on the replica but the primary tries to update it)
- Object already exists or does not exist

**Always investigate the error first.** Skipping blindly can cause data drift between primary and replica. Only skip if you understand why the error occurred and the data is acceptable on the replica.

## Step 1 - Identify the Error

```sql
SHOW REPLICA STATUS\G
```

Look for:

```text
Last_SQL_Errno: 1062
Last_SQL_Error: Could not execute Write_rows event on table mydb.users;
  Duplicate entry '42' for key 'PRIMARY'
```

## Method 1 - Skip Using GTID Mode (Recommended)

If your setup uses GTIDs, inject an empty transaction to skip the problematic event:

**Find the failing GTID:**

```sql
SHOW REPLICA STATUS\G
```

```text
Retrieved_Gtid_Set: uuid:1-100
Executed_Gtid_Set:  uuid:1-98
```

The next expected transaction is `uuid:99`.

**Inject an empty transaction for the failing GTID:**

```sql
STOP REPLICA;
SET GTID_NEXT = 'uuid:99';
BEGIN;
COMMIT;
SET GTID_NEXT = 'AUTOMATIC';
START REPLICA;
```

Replace `uuid:99` with the actual GTID from your output.

## Method 2 - Skip Using sql_replica_skip_counter (Non-GTID)

If you are not using GTIDs, use the skip counter:

```sql
STOP REPLICA SQL_THREAD;
SET GLOBAL sql_replica_skip_counter = 1;
START REPLICA SQL_THREAD;
```

This skips the next one event from the binary log. For older MySQL versions:

```sql
SET GLOBAL sql_slave_skip_counter = 1;
```

To skip multiple events (use with caution):

```sql
SET GLOBAL sql_replica_skip_counter = 5;
```

## Step 3 - Verify Replication Resumed

```sql
SHOW REPLICA STATUS\G
```

Confirm:

```text
Replica_IO_Running: Yes
Replica_SQL_Running: Yes
Last_SQL_Errno: 0
Last_SQL_Error:
```

## Configuring Automatic Error Skipping (Not Recommended for Production)

You can configure specific error codes to be skipped automatically:

```ini
[mysqld]
replica_skip_errors = 1062,1032
```

- `1062` - Duplicate entry
- `1032` - Row not found

This is a blunt tool and can mask real data integrity issues. Use it only for temporary mitigation.

## Reconciling Data After Skipping

After skipping an error, check if the data is consistent:

```sql
-- On the primary
SELECT * FROM mydb.users WHERE id = 42;

-- On the replica
SELECT * FROM mydb.users WHERE id = 42;
```

If they differ, manually reconcile the row:

```sql
-- On the replica (fix to match primary)
UPDATE mydb.users SET name = 'John', email = 'john@example.com' WHERE id = 42;
```

For large-scale consistency checks, use `pt-table-checksum` from Percona Toolkit.

## Summary

To skip a replication error in GTID mode, inject an empty transaction using `SET GTID_NEXT` to advance past the failing event. Without GTIDs, use `SET GLOBAL sql_replica_skip_counter = 1`. Always verify the data on both servers after skipping to prevent silent data divergence.
