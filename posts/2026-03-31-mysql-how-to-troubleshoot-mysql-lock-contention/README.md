# How to Troubleshoot MySQL Lock Contention

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Locks, InnoDB, Deadlocks, Performance, Troubleshooting

Description: Learn how to identify and resolve MySQL lock contention, deadlocks, and long-running transactions using Performance Schema and InnoDB diagnostics.

---

## Types of Locks in MySQL InnoDB

InnoDB uses several lock types:
- **Row-level locks** - S (shared) and X (exclusive) locks on individual rows.
- **Gap locks** - lock the gap between rows to prevent phantom reads.
- **Next-key locks** - combination of row lock and gap lock (default in REPEATABLE READ).
- **Table-level locks** - MDL (metadata locks) during DDL operations.
- **Intention locks** - IS/IX locks on tables indicating row-level locking intent.

## Finding Current Locks

### Using InnoDB Status

```sql
SHOW ENGINE INNODB STATUS\G
```

Look for the `TRANSACTIONS` section. It shows waiting and holding transactions:

```text
---TRANSACTION 421908, ACTIVE 30 sec starting index read
MySQL thread id 15, OS thread handle 140..., query id 123 localhost app_user updating
UPDATE orders SET status = 'processed' WHERE customer_id = 42
------- TRX HAS BEEN WAITING 30 sec FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 78 page no 4 n bits 72 index PRIMARY of table `myapp`.`orders` ...
```

### Using Performance Schema

```sql
-- View current data locks
SELECT
  r.trx_id AS waiting_trx_id,
  r.trx_mysql_thread_id AS waiting_thread,
  r.trx_query AS waiting_query,
  b.trx_id AS blocking_trx_id,
  b.trx_mysql_thread_id AS blocking_thread,
  b.trx_query AS blocking_query
FROM information_schema.innodb_lock_waits w
JOIN information_schema.innodb_trx r ON r.trx_id = w.requesting_trx_id
JOIN information_schema.innodb_trx b ON b.trx_id = w.blocking_trx_id;
```

In MySQL 8, use `performance_schema`:

```sql
SELECT
  wt.THREAD_ID AS waiting_thread,
  wt.PROCESSLIST_INFO AS waiting_query,
  bt.THREAD_ID AS blocking_thread,
  bt.PROCESSLIST_INFO AS blocking_query,
  l.LOCK_TYPE,
  l.LOCK_MODE,
  l.LOCK_STATUS,
  l.OBJECT_NAME AS table_name
FROM performance_schema.data_lock_waits dlw
JOIN performance_schema.data_locks l ON l.ENGINE_LOCK_ID = dlw.BLOCKING_ENGINE_LOCK_ID
JOIN performance_schema.threads wt ON wt.THREAD_ID = dlw.REQUESTING_THREAD_ID
JOIN performance_schema.threads bt ON bt.THREAD_ID = dlw.BLOCKING_THREAD_ID;
```

### View All Active Transactions

```sql
SELECT
  trx_id,
  trx_state,
  trx_started,
  TIMESTAMPDIFF(SECOND, trx_started, NOW()) AS trx_age_seconds,
  trx_rows_locked,
  trx_rows_modified,
  trx_query
FROM information_schema.innodb_trx
ORDER BY trx_age_seconds DESC;
```

## Finding and Killing the Blocking Process

```sql
-- Get the blocking thread's process ID
SELECT p.ID, p.USER, p.HOST, p.DB, p.COMMAND, p.TIME, p.STATE, p.INFO
FROM information_schema.PROCESSLIST p
WHERE p.ID IN (
  SELECT b.trx_mysql_thread_id
  FROM information_schema.innodb_trx b
  JOIN information_schema.innodb_lock_waits w ON b.trx_id = w.blocking_trx_id
);

-- Kill the blocking connection
KILL CONNECTION 15;
```

## Diagnosing Deadlocks

InnoDB automatically detects deadlocks and rolls back the transaction with the least work. View the last deadlock:

```sql
SHOW ENGINE INNODB STATUS\G
-- Look for "LATEST DETECTED DEADLOCK"
```

### Enable Deadlock Logging

```sql
SET GLOBAL innodb_print_all_deadlocks = ON;
```

This logs all deadlocks to the MySQL error log - useful for pattern analysis.

### Common Deadlock Pattern - Reverse Lock Order

```sql
-- Transaction 1
START TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;  -- Locks row 1
UPDATE accounts SET balance = balance + 100 WHERE id = 2;  -- Waits for row 2

-- Transaction 2 (concurrent)
START TRANSACTION;
UPDATE accounts SET balance = balance - 50 WHERE id = 2;   -- Locks row 2
UPDATE accounts SET balance = balance + 50 WHERE id = 1;   -- Waits for row 1 -> DEADLOCK
```

**Fix**: Always access rows in the same order across transactions.

## Reducing Lock Contention

### Use Shorter Transactions

```sql
-- Bad: long transaction holds locks
START TRANSACTION;
SELECT * FROM large_table FOR UPDATE;
-- ... application processing for 10 seconds ...
UPDATE large_table SET ...;
COMMIT;

-- Better: minimize time between SELECT and UPDATE
SELECT id FROM large_table WHERE condition = 1;
-- Process the IDs in application
START TRANSACTION;
UPDATE large_table SET ... WHERE id IN (...);
COMMIT;
```

### Use SELECT ... FOR UPDATE Wisely

```sql
-- Only lock the rows you need
SELECT id FROM orders WHERE status = 'pending' LIMIT 10 FOR UPDATE SKIP LOCKED;
```

`SKIP LOCKED` (MySQL 8) skips rows already locked by other transactions - useful for job queue patterns.

### Check and Tune innodb_lock_wait_timeout

```sql
SHOW VARIABLES LIKE 'innodb_lock_wait_timeout';
SET GLOBAL innodb_lock_wait_timeout = 30;  -- Default 50 seconds
```

### Monitor Lock Wait Metrics

```sql
SHOW STATUS LIKE 'Innodb_row_lock%';
```

Key metrics:
- `Innodb_row_lock_waits` - total number of times a lock wait occurred.
- `Innodb_row_lock_time_avg` - average time waiting for a lock in milliseconds.
- `Innodb_row_lock_time_max` - maximum time waiting.

## Summary

MySQL lock contention diagnosis starts with `SHOW ENGINE INNODB STATUS` and the `performance_schema.data_lock_waits` table to identify blocking and waiting transactions. Kill long-running blocking sessions that hold locks unnecessarily, always access rows in a consistent order to avoid deadlocks, use `SKIP LOCKED` for queue workloads, and tune `innodb_lock_wait_timeout` to control how long transactions wait before aborting.
