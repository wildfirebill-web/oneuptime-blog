# How to Start a Transaction in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Transaction, InnoDB, Database, ACID

Description: Learn how to start a transaction in MySQL using START TRANSACTION and BEGIN, and understand autocommit behavior and transaction scope.

---

## Overview

A transaction in MySQL groups multiple SQL statements into a single atomic unit - either all statements succeed (COMMIT) or all are rolled back (ROLLBACK). Transactions ensure data consistency under concurrent access and system failures. MySQL uses InnoDB as its transactional storage engine.

## Starting a Transaction

MySQL provides two equivalent ways to start a transaction:

```sql
-- Method 1: START TRANSACTION (SQL standard)
START TRANSACTION;

-- Method 2: BEGIN (shorthand)
BEGIN;
```

Both initiate a new transaction. Subsequent SQL statements are part of that transaction until you issue `COMMIT` or `ROLLBACK`.

## A Basic Transaction Example

```sql
START TRANSACTION;

UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;

COMMIT;
```

Both updates are applied atomically. If the server crashes before COMMIT, neither update persists.

## Autocommit Mode

By default, MySQL operates in autocommit mode - each statement is automatically committed:

```sql
-- Check autocommit status
SELECT @@autocommit;

-- In autocommit mode, each statement auto-commits:
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
-- This is immediately committed
```

`START TRANSACTION` temporarily disables autocommit for the duration of that transaction, regardless of the global autocommit setting.

## Disabling Autocommit Globally

You can disable autocommit for the session, requiring explicit COMMIT after each operation:

```sql
SET autocommit = 0;

UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;

COMMIT;
```

Re-enable autocommit when done:

```sql
SET autocommit = 1;
```

## START TRANSACTION WITH Options

MySQL allows read-only and consistent snapshot options:

```sql
-- Read-only transaction: prevents accidental writes
START TRANSACTION READ ONLY;

SELECT * FROM accounts;
-- Attempting writes will fail with an error

COMMIT;

-- Consistent snapshot: sees data as of transaction start
START TRANSACTION WITH CONSISTENT SNAPSHOT;
SELECT * FROM large_reporting_table;
COMMIT;
```

`WITH CONSISTENT SNAPSHOT` is useful for long-running reports that need a consistent view of data without locking rows.

## Transaction Scope

Transactions span multiple statements and can include any DML (INSERT, UPDATE, DELETE) and SELECT statements. DDL statements (CREATE TABLE, ALTER TABLE) cause an implicit commit:

```sql
START TRANSACTION;
INSERT INTO orders (customer_id, total) VALUES (1, 250.00);
-- Implicit commit caused by DDL - cannot roll back the INSERT above
CREATE TABLE temp_log (id INT);
ROLLBACK; -- Only affects statements after the implicit commit
```

## Checking Active Transactions

```sql
SELECT * FROM information_schema.INNODB_TRX;
```

This shows all active InnoDB transactions, their start time, and current SQL.

## Monitoring Transaction Activity with OneUptime

Long-running open transactions are a common cause of lock contention and connection pool exhaustion. Use OneUptime to monitor database connection counts and query latency, and alert when thresholds indicate stuck transactions.

## Summary

Start a MySQL transaction with `START TRANSACTION` or `BEGIN`. MySQL's autocommit mode commits each statement individually by default - explicit transactions override this. Use `READ ONLY` for report queries and `WITH CONSISTENT SNAPSHOT` for long-running reads. Always end transactions with COMMIT or ROLLBACK to release locks promptly.
