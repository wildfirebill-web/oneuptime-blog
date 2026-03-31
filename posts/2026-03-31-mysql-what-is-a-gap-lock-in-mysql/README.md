# What Is a Gap Lock in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Gap Lock, InnoDB, Locking, Transaction, Phantom Reads

Description: Learn what a MySQL InnoDB gap lock is, when it is acquired, how it prevents phantom reads, and how it can cause unexpected deadlocks.

---

## What Is a Gap Lock

A gap lock in InnoDB is a lock placed on the gap between two indexed values, not on an actual row. It prevents other transactions from inserting new rows into that gap, which would otherwise result in phantom reads.

Gap locks are part of InnoDB's strategy for implementing REPEATABLE READ isolation. They protect range queries from seeing newly inserted rows that would match the query's WHERE clause if the query were re-executed.

## Why Gap Locks Exist

Consider a transaction that runs:

```sql
BEGIN;
SELECT * FROM orders WHERE status = 'pending' AND total > 100 FOR UPDATE;
```

Without gap locks, another transaction could insert a new order with `status = 'pending'` and `total = 150` before the first transaction commits. If the first transaction then re-runs the query, it would see the new row - a phantom read.

A gap lock prevents this insertion by locking the gap where the new row would be inserted.

## When InnoDB Acquires Gap Locks

Gap locks are acquired when:
- A range query uses `SELECT ... FOR UPDATE` or `SELECT ... FOR SHARE`
- A range DELETE or UPDATE is executed
- The query uses a non-unique index

```sql
-- This query acquires a gap lock on the gap around index values 10-20
SELECT * FROM products WHERE price BETWEEN 10 AND 20 FOR UPDATE;

-- This range DELETE acquires gap locks
DELETE FROM sessions WHERE expires_at < '2026-01-01';
```

## Gap Locks Do Not Conflict with Each Other

Unlike record locks, two transactions can both hold gap locks on the same gap simultaneously. Gap locks only conflict with INSERT operations that would insert into the locked gap.

```sql
-- Session 1: acquires gap lock on gap (5, 10)
SELECT * FROM employees WHERE department_id BETWEEN 5 AND 10 FOR UPDATE;

-- Session 2: can also acquire gap lock on the same gap
SELECT * FROM employees WHERE department_id BETWEEN 5 AND 10 FOR UPDATE;

-- But this INSERT from either session or a third session would be blocked:
INSERT INTO employees (department_id, name) VALUES (7, 'Alice');
```

## Visualizing Gap Locks

Given an index with values: `[1, 5, 10, 15, 20]`

A query filtering on `id > 5 AND id < 15` would lock the gaps:
- Gap between 5 and 10
- Gap between 10 and 15

```text
Index values:  1 ... 5 ... (gap) ... 10 ... (gap) ... 15 ... 20
                        ^-- locked --^          ^-- locked --^
```

No INSERT of a value in these gaps would be allowed while the lock is held.

## Detecting Gap Locks

View current gap locks in the InnoDB lock monitor:

```sql
SHOW ENGINE INNODB STATUS\G
```

Look for `GAP` in the lock section:

```text
RECORD LOCKS space id 45 page no 3 n bits 72 index idx_status of table myapp.orders
trx id 12345 lock_mode X locks gap before rec
```

You can also query the Performance Schema:

```sql
SELECT
  ENGINE_LOCK_ID,
  ENGINE_TRANSACTION_ID,
  LOCK_TYPE,
  LOCK_MODE,
  LOCK_STATUS,
  LOCK_DATA
FROM performance_schema.data_locks
WHERE LOCK_MODE LIKE '%GAP%';
```

## Gap Locks and Deadlocks

Gap locks can cause unexpected deadlocks when two transactions try to insert into the same gap:

```sql
-- Session 1: SELECT creates gap lock on gap (5, 10)
BEGIN;
SELECT * FROM products WHERE id BETWEEN 5 AND 10 FOR UPDATE;

-- Session 2: SELECT also acquires gap lock on same gap
BEGIN;
SELECT * FROM products WHERE id BETWEEN 5 AND 10 FOR UPDATE;

-- Session 1 tries to INSERT into the gap - blocked by Session 2's gap lock
INSERT INTO products (id, name) VALUES (7, 'Widget');

-- Session 2 tries to INSERT into the gap - blocked by Session 1's gap lock
INSERT INTO products (id, name) VALUES (8, 'Gadget');

-- DEADLOCK! Both sessions are waiting for each other.
```

## Disabling Gap Locks

Gap locks are only used at REPEATABLE READ and SERIALIZABLE isolation levels. Switching to READ COMMITTED disables gap locks:

```sql
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

At READ COMMITTED, phantom reads are possible but gap lock contention and related deadlocks disappear.

Gap locks can also be disabled by making the query use a unique index (exact match on a unique key never requires a gap lock):

```sql
-- Uses record lock only (unique index, exact match)
SELECT * FROM orders WHERE id = 42 FOR UPDATE;
```

## Summary

A MySQL gap lock is placed on the space between indexed values to prevent phantom reads in REPEATABLE READ transactions. Gap locks block INSERT operations into the protected gap but do not conflict with other gap locks. They are essential for snapshot consistency in range queries but can be a source of unexpected deadlocks when multiple transactions compete to insert into the same gap. Switching to READ COMMITTED isolation or using exact unique-index lookups eliminates gap lock contention.
