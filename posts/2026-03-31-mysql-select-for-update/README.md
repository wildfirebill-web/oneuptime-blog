# How to Use SELECT ... FOR UPDATE in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, InnoDB, Locking, Transaction, Concurrency

Description: Learn how SELECT ... FOR UPDATE acquires exclusive row locks in MySQL to safely read and then modify rows within a transaction.

---

## What Is SELECT ... FOR UPDATE?

`SELECT ... FOR UPDATE` is a locking read that acquires an exclusive (X) lock on the selected rows. No other transaction can read (with a locking read), update, or delete those rows until the current transaction commits or rolls back. It is used to read rows that you intend to update, preventing lost updates.

## Basic Syntax

```sql
SELECT column1, column2
FROM table_name
WHERE condition
FOR UPDATE;
```

This must be executed inside a transaction to be meaningful:

```sql
START TRANSACTION;
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;
-- Row is now exclusively locked
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
COMMIT;
```

## Preventing Lost Updates

The classic use case is preventing a race condition in read-modify-write operations:

```sql
-- Without FOR UPDATE: two sessions read balance = 1000 simultaneously,
-- both subtract 100, both write 900 - one update is lost!

-- With FOR UPDATE: safe pattern
START TRANSACTION;
SELECT balance FROM accounts WHERE id = 1 FOR UPDATE;
-- Guaranteed: no other session can modify this row now

UPDATE accounts SET balance = balance - 100 WHERE id = 1;
COMMIT;
```

## FOR UPDATE with Subqueries

```sql
-- Lock rows returned by a subquery
START TRANSACTION;
SELECT * FROM orders
WHERE customer_id IN (
    SELECT id FROM customers WHERE tier = 'premium'
)
FOR UPDATE;
COMMIT;
```

Note: the subquery rows (customers) are not locked - only the rows from the outer query (orders) are locked.

## SKIP LOCKED and NOWAIT Options

MySQL 8.0 added options to avoid waiting for locked rows:

```sql
-- SKIP LOCKED: skip rows that are currently locked (useful for job queues)
START TRANSACTION;
SELECT * FROM jobs
WHERE status = 'pending'
ORDER BY created_at
LIMIT 1
FOR UPDATE SKIP LOCKED;
COMMIT;
```

```sql
-- NOWAIT: return an error immediately instead of waiting
START TRANSACTION;
SELECT * FROM accounts WHERE id = 1 FOR UPDATE NOWAIT;
-- Raises ERROR 3572 if the row is already locked
COMMIT;
```

SKIP LOCKED is commonly used to implement efficient work queues where multiple workers pull tasks without contention.

## Locking Behavior with Indexes

The lock scope depends on the index used:

```sql
-- Primary key: record lock only on the specific row
SELECT * FROM orders WHERE id = 5 FOR UPDATE;

-- Non-unique secondary index: next-key locks (record + gap)
SELECT * FROM orders WHERE status = 'pending' FOR UPDATE;

-- No index: full table scan with locks on all rows
SELECT * FROM orders WHERE notes = 'urgent' FOR UPDATE;
-- Avoid this pattern - use proper indexes
```

## Checking Locks Held by FOR UPDATE

```sql
SELECT LOCK_MODE, LOCK_TYPE, LOCK_DATA, OBJECT_NAME
FROM performance_schema.data_locks
WHERE LOCK_MODE IN ('X', 'X,REC_NOT_GAP');
```

## Releasing the Lock

Locks acquired by FOR UPDATE are released when the transaction ends:

```sql
-- Released on COMMIT
COMMIT;

-- Also released on ROLLBACK
ROLLBACK;
```

## Summary

`SELECT ... FOR UPDATE` acquires exclusive locks on selected rows, preventing any other transaction from reading (via locking reads), updating, or deleting them until the transaction finishes. Use it for read-modify-write patterns to prevent lost updates. The SKIP LOCKED and NOWAIT options provide non-blocking alternatives suitable for queue processing and fast-fail scenarios.
