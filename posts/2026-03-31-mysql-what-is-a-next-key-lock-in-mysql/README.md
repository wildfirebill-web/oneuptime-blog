# What Is a Next-Key Lock in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Next-Key Lock, InnoDB, Locking, REPEATABLE READ, Concurrency

Description: Learn what a MySQL InnoDB next-key lock is, how it combines a record lock and a gap lock, and when InnoDB uses it to prevent phantom reads.

---

## What Is a Next-Key Lock

A next-key lock is InnoDB's default row-level locking mechanism at REPEATABLE READ isolation. It is a combination of a record lock on an index row and a gap lock on the gap immediately before that index row. Together they lock a range that includes the record itself and the space before it, preventing both modification of the existing record and insertion of new records into the gap.

The term "next-key" refers to locking the key that comes next in the index, along with the gap preceding it.

## How Next-Key Locks Work

Given an index with values `[10, 20, 30, 40]`, InnoDB divides the index into intervals called "next-key ranges":

```text
(-infinity, 10]
(10, 20]
(20, 30]
(30, 40]
(40, +infinity)
```

When a transaction scans the index and finds a matching row, InnoDB places a next-key lock on the interval ending at that row. This includes locking the row itself (record lock) and the gap before it.

## Example: Next-Key Lock Behavior

```sql
-- Create a table with some data
CREATE TABLE accounts (
  id INT PRIMARY KEY,
  owner VARCHAR(50),
  balance DECIMAL(10,2),
  INDEX idx_balance (balance)
);

INSERT INTO accounts VALUES (1, 'Alice', 100.00);
INSERT INTO accounts VALUES (2, 'Bob', 200.00);
INSERT INTO accounts VALUES (3, 'Charlie', 300.00);
```

When Session 1 runs:

```sql
BEGIN;
SELECT * FROM accounts WHERE balance = 200.00 FOR UPDATE;
```

InnoDB acquires:
- A next-key lock on `(100.00, 200.00]` (gap before 200 plus the 200 record)
- Possibly a gap lock on `(200.00, 300.00)` to prevent new rows with balance between 200 and 300

This prevents Session 2 from inserting a row with balance 150 or 250 while Session 1 is active.

## Next-Key Locks vs Gap Locks vs Record Locks

```text
Lock Type       Covers
---------       ------
Record lock     A single index row
Gap lock        The gap before an index row (no row)
Next-key lock   A single index row plus the gap before it
```

A next-key lock is the combination: `gap lock + record lock`. When InnoDB says it uses "next-key locking," it means that for every row touched by a query, it locks both the row and the preceding gap.

## Viewing Next-Key Locks

Check active locks in the Performance Schema:

```sql
SELECT
  ENGINE_LOCK_ID,
  ENGINE_TRANSACTION_ID,
  LOCK_TYPE,
  LOCK_MODE,
  LOCK_DATA
FROM performance_schema.data_locks
WHERE LOCK_TYPE = 'RECORD';
```

Lock modes containing `X` with no additional modifier are typically next-key locks. Lock modes containing `X,GAP` are gap-only locks.

From the InnoDB status output:

```sql
SHOW ENGINE INNODB STATUS\G
```

Look for entries like:

```text
RECORD LOCKS space id 45 page no 4 n bits 80 index idx_balance of table myapp.accounts
trx id 9876 lock_mode X
```

## When InnoDB Uses Next-Key Locks

InnoDB uses next-key locks by default at REPEATABLE READ when:
- A query scans a non-unique index
- A range query is executed with FOR UPDATE or FOR SHARE
- A range DELETE or UPDATE is performed

When the query matches on a unique index with an exact value, InnoDB uses only a record lock (no gap lock is needed because no phantom can occur with a unique key):

```sql
-- Only a record lock (unique index, exact match)
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;
```

## Next-Key Locks and Phantom Prevention

The primary purpose of next-key locks is phantom prevention. A phantom is a row that appears in a re-executed range query because it was inserted by another transaction after the first read.

```sql
-- Session 1: range query with FOR UPDATE acquires next-key locks
BEGIN;
SELECT * FROM accounts WHERE balance > 100 AND balance < 300 FOR UPDATE;

-- Session 2: cannot insert into the locked range
INSERT INTO accounts VALUES (4, 'Diana', 150.00);  -- Blocked!
```

Session 2 is blocked because Session 1 holds next-key locks on the range `(100, 300]`, preventing the insert.

## Reducing Next-Key Lock Contention

Switch to READ COMMITTED isolation to disable gap locking entirely:

```sql
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

At READ COMMITTED, only record locks are used. Gap locks and next-key locks are not acquired. Phantom reads become possible but lock contention decreases significantly, which can improve concurrency in write-heavy workloads.

## Summary

A next-key lock is InnoDB's default row lock at REPEATABLE READ - it locks an index record plus the gap before it, preventing both modification of the record and insertion into the gap. Next-key locks are InnoDB's mechanism for preventing phantom reads in range queries. They operate automatically and transparently at the storage engine level. If gap lock contention is causing performance issues, switching to READ COMMITTED isolation removes gap and next-key locks at the cost of allowing phantom reads.
