# What Is a Transaction Isolation Level in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Transaction Isolation, InnoDB, MVCC, Concurrency, ACID

Description: Learn what transaction isolation levels are in MySQL, how they control concurrency anomalies, and how to choose the right level for your workload.

---

## What Is Transaction Isolation

Transaction isolation defines how changes made within a transaction are visible to other concurrent transactions. It controls the tradeoff between data consistency and concurrency performance. SQL standard defines four isolation levels, and MySQL supports all four for InnoDB tables.

Higher isolation levels provide stronger consistency guarantees but increase the chance of lock contention and reduce throughput.

## The Four Isolation Levels

```text
Level                  Dirty Read   Non-Repeatable Read   Phantom Read
----                   ----------   -------------------   ------------
READ UNCOMMITTED       Possible     Possible              Possible
READ COMMITTED         Prevented    Possible              Possible
REPEATABLE READ        Prevented    Prevented             Possible (prevented in InnoDB)
SERIALIZABLE           Prevented    Prevented             Prevented
```

## READ UNCOMMITTED

At this level, a transaction can read data that has been modified by another transaction that has not yet committed. These are called dirty reads.

```sql
-- Session 1: start a transaction and update a row
BEGIN;
UPDATE accounts SET balance = 1000 WHERE id = 1;
-- Not committed yet

-- Session 2 (READ UNCOMMITTED): can see the uncommitted change
SET SESSION TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
BEGIN;
SELECT balance FROM accounts WHERE id = 1;  -- Returns 1000 (dirty read)
```

READ UNCOMMITTED is almost never used in production because dirty reads lead to data integrity issues.

## READ COMMITTED

A transaction can only see data that has been committed. Dirty reads are prevented, but if another transaction commits a change between two reads within the same transaction, the second read may return different results (non-repeatable read).

```sql
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- Session 1:
BEGIN;
SELECT balance FROM accounts WHERE id = 1;  -- Returns 500

-- Session 2 commits an update:
UPDATE accounts SET balance = 800 WHERE id = 1;
COMMIT;

-- Session 1 reads again:
SELECT balance FROM accounts WHERE id = 1;  -- Returns 800 (different from first read)
COMMIT;
```

READ COMMITTED is the default for PostgreSQL and is widely used when you need up-to-date reads and can tolerate non-repeatable reads.

## REPEATABLE READ

Within a transaction, all reads return the same snapshot of data as the first read. InnoDB uses MVCC to provide this without blocking writers. This is the default isolation level in MySQL.

```sql
-- MySQL default
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;

BEGIN;
SELECT balance FROM accounts WHERE id = 1;  -- Returns 500

-- Another transaction commits balance = 800
-- But Session 1 still sees 500:
SELECT balance FROM accounts WHERE id = 1;  -- Still returns 500

COMMIT;
-- After commit, new reads see the latest committed data
```

In standard SQL, REPEATABLE READ still allows phantom reads (new rows inserted by another transaction appearing in a range scan). InnoDB prevents phantoms using gap locks and next-key locks.

## SERIALIZABLE

The strictest isolation level. All transactions appear to execute serially one after another. InnoDB converts all plain SELECT statements to `SELECT ... FOR SHARE`, adding shared locks that prevent other transactions from modifying the locked rows.

```sql
SET SESSION TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

SERIALIZABLE eliminates all concurrency anomalies but significantly reduces throughput due to increased locking.

## Checking and Setting Isolation Levels

Check the current isolation level:

```sql
-- MySQL 8.0+
SELECT @@transaction_isolation;

-- Older versions
SELECT @@tx_isolation;
```

Set for the current session:

```sql
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
SET SESSION TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
SET SESSION TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

Set globally for all new connections:

```sql
SET GLOBAL TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

Set in `my.cnf`:

```ini
[mysqld]
transaction_isolation = READ-COMMITTED
```

## Which Level Should You Use

For most web applications, REPEATABLE READ (MySQL default) is appropriate. It provides snapshot consistency within a transaction without the overhead of SERIALIZABLE.

READ COMMITTED is preferred when:
- You want to see the latest committed data on every read
- Your application is read-heavy and you want to minimize gap lock contention
- You use row-based binary logging (required for some multi-statement transactions)

SERIALIZABLE is appropriate for financial systems where strict ordering guarantees are required.

## Summary

MySQL supports four transaction isolation levels: READ UNCOMMITTED, READ COMMITTED, REPEATABLE READ (default), and SERIALIZABLE. Each level controls which concurrency anomalies - dirty reads, non-repeatable reads, and phantom reads - are permitted. InnoDB's MVCC implementation allows high concurrency even at REPEATABLE READ by serving consistent snapshots without blocking reads, making it the right default for most production MySQL workloads.
