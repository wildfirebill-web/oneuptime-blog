# How to Use ROLLBACK TO SAVEPOINT in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Transactions, Savepoints, Data Integrity

Description: Learn how to use ROLLBACK TO SAVEPOINT in MySQL to selectively undo part of a transaction while preserving earlier changes.

---

## What Is ROLLBACK TO SAVEPOINT?

`ROLLBACK TO SAVEPOINT` undoes all changes made since the named savepoint was created, but does not end the transaction. The transaction remains active and you can continue with more statements, then either `COMMIT` or issue another `ROLLBACK`.

## Syntax

```sql
ROLLBACK TO SAVEPOINT savepoint_name;
-- or shorthand (MySQL also accepts):
ROLLBACK TO savepoint_name;
```

## Basic Example

```sql
BEGIN;

UPDATE accounts SET balance = balance - 100 WHERE account_id = 1;

SAVEPOINT after_debit;

UPDATE accounts SET balance = balance + 100 WHERE account_id = 2;

-- The second update had wrong data - undo just that
ROLLBACK TO SAVEPOINT after_debit;

-- Continue with the correct amount
UPDATE accounts SET balance = balance + 100 WHERE account_id = 3;

COMMIT;
```

After `COMMIT`, only the debit from account 1 and the credit to account 3 are saved.

## What ROLLBACK TO SAVEPOINT Does NOT Do

It does NOT end the transaction. It does NOT commit earlier changes. The transaction remains open:

```sql
BEGIN;

INSERT INTO log (msg) VALUES ('entry 1');
SAVEPOINT sp1;

INSERT INTO log (msg) VALUES ('entry 2');

ROLLBACK TO SAVEPOINT sp1;  -- 'entry 2' undone, 'entry 1' still pending

-- Transaction is still active - must still COMMIT or ROLLBACK
SELECT * FROM log;  -- 'entry 1' visible in this session (uncommitted)

COMMIT;  -- 'entry 1' permanently saved
```

## Multiple Savepoints and Selective Rollback

```sql
BEGIN;

INSERT INTO orders (user_id, amount) VALUES (1, 50);
SAVEPOINT sp_order1;

INSERT INTO orders (user_id, amount) VALUES (2, 75);
SAVEPOINT sp_order2;

INSERT INTO orders (user_id, amount) VALUES (3, 30);
SAVEPOINT sp_order3;

-- Roll back to sp_order2: undoes sp_order3's insert
ROLLBACK TO SAVEPOINT sp_order2;
-- sp_order2 still exists; sp_order3 is gone

-- Roll back to sp_order1: undoes sp_order2's insert as well
ROLLBACK TO SAVEPOINT sp_order1;

COMMIT;
-- Only the first INSERT (user_id=1) is committed
```

## Error if Savepoint Does Not Exist

```sql
ROLLBACK TO SAVEPOINT nonexistent_sp;
-- ERROR 1305 (42000): SAVEPOINT nonexistent_sp does not exist
```

Always ensure the savepoint was created before attempting to roll back to it.

## Practical Use - Batch Import with Per-Row Error Handling

```sql
BEGIN;

-- Import row 1
INSERT INTO products (sku, name, price) VALUES ('A001', 'Widget', 9.99);
SAVEPOINT after_row1;

-- Import row 2 (might violate a unique key)
INSERT INTO products (sku, name, price) VALUES ('A001', 'Duplicate Widget', 5.00);
-- If this fails, roll back only row 2
ROLLBACK TO SAVEPOINT after_row1;

-- Import row 3
INSERT INTO products (sku, name, price) VALUES ('A002', 'Gadget', 14.99);
SAVEPOINT after_row3;

COMMIT;
-- Only rows 1 and 3 are committed
```

In application code, catch the error after each insert and roll back to the previous savepoint.

## Python Implementation

```python
import mysql.connector

def bulk_import(rows):
    conn = mysql.connector.connect(host='localhost', user='root', password='pass', database='myapp')
    cursor = conn.cursor()
    conn.start_transaction()

    for i, row in enumerate(rows):
        sp_name = f"sp_row_{i}"
        cursor.execute(f"SAVEPOINT {sp_name}")
        try:
            cursor.execute(
                "INSERT INTO products (sku, name, price) VALUES (%s, %s, %s)",
                (row['sku'], row['name'], row['price'])
            )
        except mysql.connector.errors.IntegrityError as e:
            cursor.execute(f"ROLLBACK TO SAVEPOINT {sp_name}")
            print(f"Skipped row {i}: {e}")

    conn.commit()
    cursor.close()
    conn.close()
```

## ROLLBACK TO SAVEPOINT vs Full ROLLBACK

```text
ROLLBACK TO SAVEPOINT name:
  - Undoes changes since the savepoint
  - Transaction remains active
  - Can continue with more statements

ROLLBACK (no savepoint):
  - Undoes ALL changes since BEGIN
  - Transaction ends
  - Connection returns to autocommit mode
```

## Summary

`ROLLBACK TO SAVEPOINT` in MySQL provides granular undo control within a transaction, allowing you to discard changes since a specific named point while preserving earlier work. The transaction stays open after rolling back to a savepoint. Use this pattern for batch operations where individual failures should not abort the entire transaction. Always follow with `COMMIT` or a full `ROLLBACK` to complete the transaction.
