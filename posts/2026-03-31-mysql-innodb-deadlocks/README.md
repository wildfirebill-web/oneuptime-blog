# How to Monitor InnoDB Deadlocks in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, InnoDB, Deadlock, Monitoring

Description: Learn how InnoDB detects and resolves deadlocks, how to read the last deadlock report, and how to configure monitoring and prevention strategies.

---

## What Is a Deadlock?

A deadlock occurs when two or more transactions are each waiting for a lock held by another, creating a circular dependency. InnoDB detects these cycles automatically and kills the transaction with the lowest cost (fewest rows modified) to break the cycle.

```text
Transaction A holds lock on row 1, wants lock on row 2
Transaction B holds lock on row 2, wants lock on row 1
  -> Deadlock detected -> Transaction B is rolled back (victim)
```

The application receives error 1213:

```text
ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction
```

## Viewing the Last Deadlock

```sql
SHOW ENGINE INNODB STATUS\G
```

Look for the `LATEST DETECTED DEADLOCK` section:

```text
LATEST DETECTED DEADLOCK
-------------------------
2026-03-31 10:00:00 UTC
*** (1) TRANSACTION:
TRANSACTION 12345, ACTIVE 0 sec starting index read
...
LOCK WAIT 2 lock struct(s), heap size 1136, 1 row lock(s)
...
*** (1) HOLDS THE LOCK(S):
RECORD LOCKS space id 42 page no 7 n bits 72 index PRIMARY of table `mydb`.`orders`
...
*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
...
*** (2) TRANSACTION:
...
*** WE ROLL BACK TRANSACTION (2)
```

## Enabling Deadlock Logging

By default, only the last deadlock is stored in memory. Enable persistent logging:

```sql
-- Log all deadlocks to the error log
SET GLOBAL innodb_print_all_deadlocks = ON;

-- In my.cnf:
-- innodb_print_all_deadlocks = ON
```

```bash
# Then tail the error log to capture deadlock details
tail -f /var/log/mysql/error.log | grep -A 50 "DEADLOCK"
```

## Monitoring via Performance Schema

```sql
-- Count deadlocks
SELECT variable_name, variable_value
FROM performance_schema.global_status
WHERE variable_name = 'Innodb_deadlocks';

-- Lock waits in progress (MySQL 8.0+)
SELECT r.trx_id waiting_trx,
       r.trx_query waiting_query,
       b.trx_id blocking_trx,
       b.trx_query blocking_query
FROM information_schema.innodb_trx r
JOIN performance_schema.data_lock_waits dlw
     ON r.trx_id = dlw.REQUESTING_ENGINE_TRANSACTION_ID
JOIN information_schema.innodb_trx b
     ON b.trx_id = dlw.BLOCKING_ENGINE_TRANSACTION_ID;
```

## Deadlock Prevention Strategies

**1. Consistent lock ordering** - always acquire locks on resources in the same order across all transactions.

```sql
-- Both transactions should always lock in the same order:
-- Lock orders row id=1 first, then id=2 (never reverse)
BEGIN;
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;
SELECT * FROM accounts WHERE id = 2 FOR UPDATE;
COMMIT;
```

**2. Keep transactions short** - reduce the window during which locks are held.

```sql
-- Bad: long transaction with external calls
BEGIN;
-- ... do API call (seconds) ...
UPDATE orders SET status = 'PROCESSED' WHERE id = 42;
COMMIT;

-- Better: minimize work inside transaction
UPDATE orders SET status = 'PROCESSED' WHERE id = 42;
```

**3. Use appropriate isolation level** - READ COMMITTED reduces gap locks, which are a common deadlock source.

```sql
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

**4. Index your WHERE clauses** - unindexed UPDATE/DELETE locks many rows via table scans.

## Deadlock Detection Configuration

```sql
-- InnoDB deadlock detection (default ON)
SHOW VARIABLES LIKE 'innodb_deadlock_detect';
-- Disabling it forces lock timeout instead (innodb_lock_wait_timeout)

-- Lock wait timeout (seconds before giving up)
SHOW VARIABLES LIKE 'innodb_lock_wait_timeout';
-- Default: 50 seconds
```

## Summary

InnoDB detects deadlocks automatically by analyzing the lock-wait graph and rolling back the cheapest transaction. Enable `innodb_print_all_deadlocks` to log every deadlock for later analysis. Prevent deadlocks by acquiring locks in a consistent global order, keeping transactions as short as possible, using READ COMMITTED isolation to reduce gap locking, and ensuring DML statements use indexed lookups rather than full scans.
