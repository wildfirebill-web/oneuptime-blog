# How to Use BEGIN, COMMIT, and ROLLBACK in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Transaction, Commit, Rollback, InnoDB

Description: Learn how to use BEGIN, COMMIT, and ROLLBACK in MySQL to manage transactions, ensure data integrity, and handle errors correctly.

---

## Overview

`BEGIN`, `COMMIT`, and `ROLLBACK` are the three core transaction control statements in MySQL. They give you explicit control over when changes are persisted or discarded, enabling atomic multi-statement operations.

## BEGIN

`BEGIN` (or `START TRANSACTION`) marks the start of a transaction. All subsequent DML statements are grouped together until a COMMIT or ROLLBACK.

```sql
BEGIN;
-- Or equivalently:
START TRANSACTION;
```

## COMMIT

`COMMIT` makes all changes made since BEGIN permanent and visible to other connections:

```sql
BEGIN;
INSERT INTO orders (customer_id, total) VALUES (42, 150.00);
UPDATE inventory SET quantity = quantity - 1 WHERE product_id = 7;
COMMIT;
-- Both changes are now permanent
```

After COMMIT, the transaction ends and autocommit resumes if it was previously enabled.

## ROLLBACK

`ROLLBACK` discards all changes made since BEGIN and restores the database to its pre-transaction state:

```sql
BEGIN;
DELETE FROM orders WHERE customer_id = 42;
-- Oops, wrong condition! Undo everything
ROLLBACK;
-- No rows were deleted
```

## Typical Usage: Error Handling Pattern

The most important use of ROLLBACK is in error handling. In application code:

```sql
BEGIN;

UPDATE accounts SET balance = balance - 500 WHERE id = 1;

-- Check if balance would go negative
SELECT balance FROM accounts WHERE id = 1;
-- If balance < 0, ROLLBACK; otherwise continue

UPDATE accounts SET balance = balance + 500 WHERE id = 2;

COMMIT;
```

In application code (Python example):

```python
try:
    cursor.execute("BEGIN")
    cursor.execute("UPDATE accounts SET balance = balance - 500 WHERE id = 1")
    cursor.execute("UPDATE accounts SET balance = balance + 500 WHERE id = 2")
    connection.commit()
except Exception as e:
    connection.rollback()
    raise e
```

## Implicit COMMIT Triggers

Certain statements automatically commit any open transaction before executing. DDL statements are the most common:

```sql
BEGIN;
INSERT INTO orders VALUES (1, 100.00);
-- This DDL causes implicit commit - the INSERT is now permanent
ALTER TABLE orders ADD COLUMN notes TEXT;
ROLLBACK;
-- Only affects statements after the implicit commit (none here)
```

Other implicit commit triggers include:
- `CREATE`, `DROP`, `RENAME`, `TRUNCATE` statements
- `LOCK TABLES`, `UNLOCK TABLES`
- `SET autocommit = 1` (if autocommit was 0)

## Verifying Transaction State

Check whether you are in an active transaction:

```sql
SELECT @@in_transaction;
-- Returns 1 if inside a transaction, 0 otherwise
```

View active transactions in InnoDB:

```sql
SELECT trx_id, trx_started, trx_state, trx_query
FROM information_schema.INNODB_TRX;
```

## COMMIT and ROLLBACK with CHAIN

MySQL supports chained transactions where COMMIT or ROLLBACK immediately starts a new transaction:

```sql
BEGIN;
INSERT INTO log_entries (message) VALUES ('First batch');
COMMIT AND CHAIN;
-- Transaction immediately restarts; no need to BEGIN again
INSERT INTO log_entries (message) VALUES ('Second batch');
COMMIT;
```

## Lock Release on COMMIT and ROLLBACK

Both COMMIT and ROLLBACK release all row locks held by the transaction. Keeping transactions short minimizes lock contention:

```sql
-- Long transaction holding locks (bad)
BEGIN;
SELECT * FROM orders WHERE status = 'pending' FOR UPDATE;
-- ... application processing taking 30 seconds ...
COMMIT;

-- Better: minimize time between lock acquisition and release
BEGIN;
SELECT id FROM orders WHERE status = 'pending' FOR UPDATE LIMIT 10;
UPDATE orders SET status = 'processing' WHERE id IN (...);
COMMIT;
```

## Summary

`BEGIN` starts a transaction, `COMMIT` persists changes, and `ROLLBACK` discards them. Always wrap multi-statement operations in transactions to ensure atomicity. Handle errors by rolling back in exception handlers, and keep transactions short to minimize lock contention. Be aware of implicit commits triggered by DDL statements.
