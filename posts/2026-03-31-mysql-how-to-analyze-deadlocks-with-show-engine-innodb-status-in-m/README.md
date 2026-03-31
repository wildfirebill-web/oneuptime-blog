# How to Analyze Deadlocks with SHOW ENGINE INNODB STATUS in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, InnoDB, Deadlock, Performance, Troubleshooting

Description: Learn how to detect, read, and resolve deadlocks in MySQL using the SHOW ENGINE INNODB STATUS command and InnoDB diagnostics.

---

## What Is a Deadlock in MySQL?

A deadlock occurs when two or more transactions are waiting for each other to release locks, creating a circular dependency that MySQL must resolve by rolling back one of the transactions. InnoDB automatically detects deadlocks and terminates the transaction with the least amount of work to undo.

Understanding deadlocks requires knowing which transactions are involved, which rows they are locking, and what SQL statements triggered the conflict.

## Running SHOW ENGINE INNODB STATUS

The primary tool for deadlock analysis is the `SHOW ENGINE INNODB STATUS` command. It outputs detailed information about InnoDB internals, including the most recent deadlock.

```sql
SHOW ENGINE INNODB STATUS\G
```

The `\G` modifier formats the output vertically, making it easier to read. The output is divided into sections: transactions, buffer pool, semaphores, and the LATEST DETECTED DEADLOCK block.

## Reading the LATEST DETECTED DEADLOCK Section

When a deadlock occurs, InnoDB records it. Look for this section in the status output:

```text
------------------------
LATEST DETECTED DEADLOCK
------------------------
2024-05-10 14:32:11 0x7f5e3c002700
*** (1) TRANSACTION:
TRANSACTION 421938, ACTIVE 2 sec starting index read
mysql tables in use 1, locked 1
LOCK WAIT 3 lock struct(s), heap size 1136, 2 row lock(s)
MySQL thread id 45, OS thread handle 140032..., query id 3291 localhost root updating
UPDATE orders SET status = 'shipped' WHERE id = 101

*** (1) HOLDS THE LOCK(S):
RECORD LOCKS space id 34 page no 4 n bits 72 index PRIMARY of table orders trx id 421938 lock_mode X locks rec but not gap

*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 35 page no 6 n bits 72 index PRIMARY of table inventory trx id 421938 lock_mode X locks rec but not gap waiting

*** (2) TRANSACTION:
TRANSACTION 421939, ACTIVE 3 sec starting index read
mysql tables in use 1, locked 1
LOCK WAIT 2 lock struct(s), heap size 1136, 1 row lock(s)
MySQL thread id 46, OS thread handle 140033..., query id 3292 localhost root updating
UPDATE inventory SET quantity = quantity - 1 WHERE product_id = 55

*** (2) HOLDS THE LOCK(S):
RECORD LOCKS space id 35 page no 6 n bits 72 index PRIMARY of table inventory trx id 421939 lock_mode X locks rec but not gap

*** (2) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 34 page no 4 n bits 72 index PRIMARY of table orders trx id 421939 lock_mode X locks rec but not gap waiting

*** WE ROLL BACK TRANSACTION (2)
```

In this example:
- Transaction 1 holds a lock on `orders` and waits for `inventory`.
- Transaction 2 holds a lock on `inventory` and waits for `orders`.
- InnoDB rolled back Transaction 2 to break the cycle.

## Enabling Detailed Deadlock Logging

For ongoing monitoring, enable InnoDB deadlock logging to the MySQL error log:

```sql
SET GLOBAL innodb_print_all_deadlocks = ON;
```

This writes every deadlock to the error log file rather than only keeping the latest one in memory. To make it persistent across restarts, add it to `my.cnf`:

```text
[mysqld]
innodb_print_all_deadlocks = ON
```

## Querying the Performance Schema for Lock Waits

Before a deadlock is resolved, you can observe lock contention in real time using Performance Schema:

```sql
SELECT
  r.trx_id AS waiting_trx,
  r.trx_mysql_thread_id AS waiting_thread,
  r.trx_query AS waiting_query,
  b.trx_id AS blocking_trx,
  b.trx_mysql_thread_id AS blocking_thread,
  b.trx_query AS blocking_query
FROM information_schema.innodb_lock_waits w
JOIN information_schema.innodb_trx r ON r.trx_id = w.requesting_trx_id
JOIN information_schema.innodb_trx b ON b.trx_id = w.blocking_trx_id;
```

In MySQL 8.0, use `performance_schema.data_lock_waits` instead:

```sql
SELECT
  OBJECT_NAME,
  INDEX_NAME,
  REQUESTING_ENGINE_TRANSACTION_ID,
  BLOCKING_ENGINE_TRANSACTION_ID
FROM performance_schema.data_lock_waits;
```

## Common Causes and Fixes

| Cause | Fix |
|---|---|
| Accessing tables in different order | Always access tables in the same order across transactions |
| Missing indexes on join/filter columns | Add indexes to reduce lock scope |
| Long-running transactions | Keep transactions short and commit early |
| Bulk updates without batching | Break updates into smaller batches |

Example: consistent table access order:

```sql
-- Always lock orders before inventory
START TRANSACTION;
SELECT * FROM orders WHERE id = 101 FOR UPDATE;
SELECT * FROM inventory WHERE product_id = 55 FOR UPDATE;
UPDATE orders SET status = 'shipped' WHERE id = 101;
UPDATE inventory SET quantity = quantity - 1 WHERE product_id = 55;
COMMIT;
```

## Simulating a Deadlock for Testing

You can reproduce deadlocks in a test environment:

```sql
-- Session 1
START TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
-- (pause here)
UPDATE accounts SET balance = balance + 100 WHERE id = 2;

-- Session 2 (run while Session 1 is paused)
START TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE id = 2;
UPDATE accounts SET balance = balance + 100 WHERE id = 1;
```

One of these sessions will receive: `ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction`.

## Summary

`SHOW ENGINE INNODB STATUS` provides a detailed record of the most recent deadlock, including the transactions, locks held, and locks waited for. Enable `innodb_print_all_deadlocks` to log all deadlocks persistently. To prevent deadlocks, keep transactions short, access tables in a consistent order, and ensure proper indexing to reduce lock contention.
