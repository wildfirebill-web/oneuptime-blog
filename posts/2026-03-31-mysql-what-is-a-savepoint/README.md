# What Is a Savepoint in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Transaction, InnoDB, Savepoint, Rollback

Description: A savepoint in MySQL marks a named point within a transaction so you can roll back to that point without abandoning the entire transaction.

---

## Overview

A savepoint is a named marker you can set inside an active transaction in MySQL. It lets you roll back part of a transaction to that marker without discarding all the work done before it. Savepoints are supported by InnoDB and are useful for complex transaction logic where partial rollback is needed.

## Creating and Using Savepoints

The basic syntax is straightforward:

```sql
SAVEPOINT savepoint_name;
ROLLBACK TO SAVEPOINT savepoint_name;
RELEASE SAVEPOINT savepoint_name;
```

Here is a practical example:

```sql
BEGIN;

INSERT INTO orders (customer_id, total) VALUES (10, 250.00);
SAVEPOINT after_order;

INSERT INTO payments (order_id, amount) VALUES (LAST_INSERT_ID(), 250.00);
SAVEPOINT after_payment;

-- Something goes wrong with the shipment record
INSERT INTO shipments (order_id, address) VALUES (LAST_INSERT_ID(), NULL);
-- Oops - address is required, let's roll back just the shipment

ROLLBACK TO SAVEPOINT after_payment;

-- Fix the shipment and try again
INSERT INTO shipments (order_id, address) VALUES (42, '123 Main St');

COMMIT;
```

In this example the order and payment remain intact while the bad shipment insert is undone.

## Rolling Back to a Savepoint vs Full Rollback

`ROLLBACK TO SAVEPOINT` undoes changes made after the savepoint but keeps the transaction open. The savepoint itself is retained and can be rolled back to again:

```sql
BEGIN;
INSERT INTO audit_log (action) VALUES ('start');
SAVEPOINT sp1;

UPDATE user_credits SET credits = credits - 50 WHERE user_id = 5;
SAVEPOINT sp2;

DELETE FROM temp_cart WHERE session_id = 'abc123';

-- Roll back just the DELETE
ROLLBACK TO SAVEPOINT sp2;
-- The UPDATE is still pending

-- Roll back all the way to sp1
ROLLBACK TO SAVEPOINT sp1;
-- Only the audit_log INSERT is still pending

COMMIT;
```

A plain `ROLLBACK` always rolls back the entire transaction and ignores all savepoints.

## Releasing Savepoints

`RELEASE SAVEPOINT` removes the savepoint marker from the transaction without rolling back. It does not commit any changes:

```sql
BEGIN;
UPDATE inventory SET qty = qty - 1 WHERE product_id = 7;
SAVEPOINT before_log;
INSERT INTO change_log (product_id, change) VALUES (7, -1);
RELEASE SAVEPOINT before_log;
-- Savepoint is gone, but both changes are still pending
COMMIT;
```

## Nested Savepoints and Naming

Savepoint names are scoped to the current transaction. If you create a savepoint with a name that already exists, the old one is silently replaced:

```sql
BEGIN;
SAVEPOINT sp;
INSERT INTO t1 VALUES (1);
SAVEPOINT sp;  -- replaces the first sp
INSERT INTO t1 VALUES (2);
ROLLBACK TO SAVEPOINT sp;  -- rolls back only the second INSERT
COMMIT;
```

## Summary

Savepoints provide fine-grained control within a transaction, allowing partial rollbacks without losing earlier work. They are particularly valuable in stored procedures and application logic where different steps may fail independently. Savepoints are only supported within explicit transactions and require a transactional storage engine like InnoDB.
