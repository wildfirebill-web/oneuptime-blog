# How to Use ROLLBACK TO SAVEPOINT in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Transaction, Savepoint, Rollback, InnoDB

Description: Learn how to use ROLLBACK TO SAVEPOINT in MySQL to partially undo transaction work without rolling back the entire transaction.

---

## Overview

`ROLLBACK TO SAVEPOINT` lets you undo part of a transaction without discarding everything. A savepoint marks a point within a transaction that you can return to if an error occurs in a later step, while keeping earlier work intact.

## Creating a Savepoint

Use `SAVEPOINT` to mark a position within an active transaction:

```sql
BEGIN;
INSERT INTO orders (customer_id, total) VALUES (1, 100.00);

SAVEPOINT after_order;

INSERT INTO order_items (order_id, product_id, quantity) VALUES (LAST_INSERT_ID(), 5, 2);
```

## Rolling Back to a Savepoint

If the second insert fails or needs to be undone:

```sql
ROLLBACK TO SAVEPOINT after_order;
```

This undoes the `order_items` insert but keeps the `orders` insert. The transaction is still open - you can continue or commit:

```sql
-- Continue with corrected data
INSERT INTO order_items (order_id, product_id, quantity) VALUES (LAST_INSERT_ID(), 7, 1);
COMMIT;
```

## Multiple Savepoints in One Transaction

You can set multiple savepoints in a single transaction:

```sql
BEGIN;

INSERT INTO batch_jobs (name) VALUES ('Job A');
SAVEPOINT job_a;

INSERT INTO batch_jobs (name) VALUES ('Job B');
SAVEPOINT job_b;

INSERT INTO batch_jobs (name) VALUES ('Job C');
SAVEPOINT job_c;

-- Job C had an error, roll back to before it
ROLLBACK TO SAVEPOINT job_b;

-- Now only Job A and Job B are in the transaction
INSERT INTO batch_jobs (name) VALUES ('Job C Corrected');
COMMIT;
```

## Rolling Back to an Earlier Savepoint

Rolling back to an earlier savepoint removes all savepoints set after it:

```sql
BEGIN;
SAVEPOINT sp1;
-- ... some work ...
SAVEPOINT sp2;
-- ... more work ...
SAVEPOINT sp3;

ROLLBACK TO SAVEPOINT sp1;
-- sp2 and sp3 are now gone; sp1 still exists
```

## Releasing a Savepoint

Use `RELEASE SAVEPOINT` to remove a savepoint without rolling back:

```sql
RELEASE SAVEPOINT after_order;
```

This does not undo any work - it simply removes the savepoint marker from the transaction. This is useful after a section of code completes successfully and you no longer need the rollback point.

## Practical Pattern: Error Handling with Savepoints

Savepoints are particularly useful in stored procedures where multiple independent operations should be attempted, with partial rollback on failure:

```sql
DELIMITER //

CREATE PROCEDURE process_batch()
BEGIN
  DECLARE EXIT HANDLER FOR SQLEXCEPTION
  BEGIN
    ROLLBACK TO SAVEPOINT batch_start;
    RESIGNAL;
  END;

  START TRANSACTION;

  -- First set of work
  INSERT INTO processed_items SELECT * FROM pending_items WHERE type = 'A';
  SAVEPOINT batch_start;

  -- Second set of work (may fail)
  INSERT INTO secondary_log SELECT * FROM pending_items WHERE type = 'B';

  COMMIT;
END //

DELIMITER ;
```

## Savepoint Naming Rules

Savepoint names:
- Are case-insensitive
- Must be unique within a transaction (creating a duplicate name replaces the old one)
- Are lost when the transaction is committed or fully rolled back

```sql
BEGIN;
SAVEPOINT step1;
-- ... work ...
SAVEPOINT step1; -- Replaces the previous step1 savepoint
```

## Summary

`ROLLBACK TO SAVEPOINT` provides partial undo within a transaction, allowing you to recover from errors in specific steps without discarding all previous work. Use multiple savepoints in long transactions with independent steps. Remember to COMMIT or fully ROLLBACK the outer transaction when done - savepoints do not end the transaction.
