# How to Use REPEATABLE READ Isolation Level in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Transaction, Isolation, InnoDB, Concurrency

Description: Understand MySQL's default REPEATABLE READ isolation level, how it uses consistent snapshots, and how it prevents non-repeatable reads.

---

## What Is REPEATABLE READ?

REPEATABLE READ is the default isolation level for MySQL InnoDB. Under this level, all reads within a transaction see a consistent snapshot of the database taken at the time of the first read. This prevents both dirty reads and non-repeatable reads. MySQL's InnoDB also largely prevents phantom reads through the use of gap locks and next-key locks.

## Confirming the Default Level

```sql
-- Check the current isolation level
SELECT @@transaction_isolation;
-- Returns: REPEATABLE-READ

-- Explicitly set it
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;
```

## Consistent Read Snapshot

InnoDB uses Multi-Version Concurrency Control (MVCC) to implement the snapshot:

```sql
-- Session 1: Start transaction (snapshot taken on first read)
START TRANSACTION;
SELECT price FROM products WHERE id = 10;
-- Returns: 50.00 - snapshot established here
```

```sql
-- Session 2: Commit a change
UPDATE products SET price = 99.99 WHERE id = 10;
COMMIT;
```

```sql
-- Session 1: Same query returns the original snapshot value
SELECT price FROM products WHERE id = 10;
-- Still returns: 50.00 - snapshot is stable

COMMIT;
```

The snapshot persists for the duration of the transaction, not just one statement.

## Gap Locks and Phantom Read Prevention

InnoDB uses gap locks to prevent phantom reads in most scenarios:

```sql
START TRANSACTION;

-- This acquires next-key locks on the range, blocking inserts into the range
SELECT * FROM orders WHERE amount BETWEEN 100 AND 200 FOR UPDATE;

-- Another session cannot insert a new row with amount = 150 until this commits
COMMIT;
```

However, plain consistent reads (without FOR UPDATE / LOCK IN SHARE MODE) do not use gap locks and rely purely on the MVCC snapshot.

## Non-Repeatable Read Prevention

```sql
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;
START TRANSACTION;

SELECT balance FROM accounts WHERE id = 1;
-- Returns: 1000

-- Another session updates and commits balance to 500

SELECT balance FROM accounts WHERE id = 1;
-- Still returns: 1000 - protected by snapshot

COMMIT;
```

The same row always returns the same value within a transaction.

## When to Use REPEATABLE READ

REPEATABLE READ is the right choice for:

- Financial transactions where stable reads are critical
- Multi-step business logic that reads and then updates rows
- Applications that need predictable, repeatable query results

```sql
-- Safe balance transfer using REPEATABLE READ
START TRANSACTION;

SELECT balance FROM accounts WHERE id = 1 FOR UPDATE;  -- Locks the row
SELECT balance FROM accounts WHERE id = 2 FOR UPDATE;

UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;

COMMIT;
```

## Limitations

- Higher lock contention than READ COMMITTED due to gap locks
- Long-running transactions hold old MVCC snapshots, causing undo log growth
- May cause deadlocks when combined with range queries

```sql
-- Check undo log growth
SHOW STATUS LIKE 'Innodb_history_list_length';
```

## Summary

REPEATABLE READ is MySQL's default isolation level, providing a stable per-transaction snapshot using MVCC and gap locks. It prevents dirty reads and non-repeatable reads and largely eliminates phantom reads, making it the safest choice for most OLTP workloads without the full serialization overhead of SERIALIZABLE.
