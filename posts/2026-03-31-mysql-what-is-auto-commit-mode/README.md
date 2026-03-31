# What Is Auto-Commit Mode in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Transaction, InnoDB, Autocommit, Connection

Description: Auto-commit mode in MySQL automatically commits each SQL statement as its own transaction, making it the default behavior for most connections.

---

## Overview

Auto-commit mode is a session-level setting in MySQL that controls whether each SQL statement is automatically wrapped in its own transaction and committed immediately. When auto-commit is enabled (the default), every statement that modifies data is committed as soon as it executes successfully. When disabled, statements accumulate in a transaction until you explicitly issue `COMMIT` or `ROLLBACK`.

## Default Behavior

MySQL enables auto-commit by default for all new connections:

```sql
-- Check current auto-commit setting
SELECT @@autocommit;
-- Returns 1 (enabled)

SHOW VARIABLES LIKE 'autocommit';
```

With auto-commit on, each DML statement behaves as if it were wrapped in a transaction:

```sql
-- These two are equivalent when autocommit = 1
UPDATE accounts SET balance = balance - 100 WHERE id = 1;

-- Equivalent to:
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
COMMIT;
```

## Disabling Auto-Commit

You can disable auto-commit for a session:

```sql
SET autocommit = 0;
-- or
SET autocommit = OFF;
```

Once disabled, you must explicitly commit or roll back changes:

```sql
SET autocommit = 0;

UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;

-- Either commit both changes
COMMIT;

-- Or roll back if something went wrong
ROLLBACK;
```

## Using BEGIN to Override Auto-Commit

Even with auto-commit enabled, you can group statements into a transaction using `BEGIN` or `START TRANSACTION`. Auto-commit is suspended until the transaction ends:

```sql
-- autocommit = 1, but this still creates a multi-statement transaction
BEGIN;
INSERT INTO orders (customer_id, total) VALUES (42, 150.00);
INSERT INTO order_items (order_id, product_id, qty) VALUES (LAST_INSERT_ID(), 7, 2);
COMMIT;
```

## Auto-Commit and Storage Engines

Auto-commit behavior applies to transactional storage engines like InnoDB. Non-transactional engines like MyISAM ignore the setting since they do not support rollback. Mixing engines in a transaction can lead to partial commits that cannot be undone.

## Performance Considerations

Auto-commit has performance implications. Each committed statement flushes to disk (depending on `innodb_flush_log_at_trx_commit`). Disabling auto-commit and batching multiple writes into a single transaction reduces the number of disk flushes and can significantly improve throughput for bulk operations:

```sql
SET autocommit = 0;
-- Insert 10,000 rows in one transaction instead of 10,000 separate transactions
INSERT INTO log_events (event, ts) VALUES ('start', NOW());
-- ... more inserts ...
COMMIT;
SET autocommit = 1;
```

## Summary

Auto-commit mode determines whether MySQL automatically commits each statement. It is enabled by default and suits simple read/write workloads where atomicity across multiple statements is not needed. For operations requiring all-or-nothing guarantees, either disable auto-commit or use explicit `BEGIN`/`COMMIT` blocks. Understanding auto-commit is fundamental to writing correct, predictable MySQL applications.
