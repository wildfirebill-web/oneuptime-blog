# How to Implement Pessimistic Locking in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Transaction, Locking, Concurrency, InnoDB

Description: Learn how to implement pessimistic locking in MySQL using SELECT FOR UPDATE to prevent concurrent modifications by holding explicit row locks.

---

## What Is Pessimistic Locking?

Pessimistic locking assumes conflicts will occur and prevents them by acquiring exclusive locks at read time. In MySQL, this is implemented with `SELECT ... FOR UPDATE`, which locks the row immediately when it is read, ensuring no other transaction can modify it before your transaction completes.

## Basic Pessimistic Locking Pattern

```sql
-- Lock the row at read time
START TRANSACTION;

SELECT balance FROM accounts WHERE id = 1 FOR UPDATE;
-- Row is now exclusively locked - no other transaction can modify it

-- Perform your business logic using the locked value
UPDATE accounts SET balance = balance - 100 WHERE id = 1;

COMMIT;
-- Lock is released
```

No other transaction can execute `UPDATE` or `SELECT ... FOR UPDATE` on this row until the first transaction commits.

## Fund Transfer with Pessimistic Locking

```sql
-- Safe fund transfer: always lock in a consistent order to prevent deadlocks
START TRANSACTION;

-- Lock the lower ID first to enforce consistent ordering
SELECT balance FROM accounts WHERE id = 1 FOR UPDATE;
SELECT balance FROM accounts WHERE id = 2 FOR UPDATE;

UPDATE accounts SET balance = balance - 500 WHERE id = 1;
UPDATE accounts SET balance = balance + 500 WHERE id = 2;

COMMIT;
```

Locking rows in a consistent order (e.g., by ascending ID) is critical to prevent deadlocks when multiple transactions transfer funds simultaneously.

## NOWAIT Option for Non-Blocking Behavior

```sql
-- Fail immediately if the row is already locked
START TRANSACTION;

SELECT * FROM jobs WHERE id = 42 FOR UPDATE NOWAIT;
-- If another transaction holds a lock on id=42, raises:
-- ERROR 3572: Statement aborted because lock(s) could not be acquired immediately

COMMIT;
```

## SKIP LOCKED for Queue Processing

```sql
-- Efficient job queue: skip rows locked by other workers
START TRANSACTION;

SELECT id, payload
FROM jobs
WHERE status = 'pending'
ORDER BY created_at
LIMIT 1
FOR UPDATE SKIP LOCKED;

-- Process the job...

UPDATE jobs SET status = 'processing' WHERE id = <fetched_id>;

COMMIT;
```

SKIP LOCKED allows multiple workers to process jobs in parallel without blocking each other.

## Pessimistic Locking in Application Code

```python
import mysql.connector

def process_order(order_id):
    conn = mysql.connector.connect(host='localhost', database='shop',
                                   user='app', password='secret')
    cursor = conn.cursor(dictionary=True)
    conn.start_transaction()

    try:
        cursor.execute(
            "SELECT * FROM orders WHERE id = %s FOR UPDATE",
            (order_id,)
        )
        order = cursor.fetchone()

        if order['status'] != 'pending':
            conn.rollback()
            return False

        cursor.execute(
            "UPDATE orders SET status = 'processing' WHERE id = %s",
            (order_id,)
        )
        conn.commit()
        return True

    except Exception as e:
        conn.rollback()
        raise e
```

## Deadlock Prevention

Deadlocks occur when two transactions each hold a lock the other needs:

```sql
-- Always acquire locks in the same order across all transactions
-- Order by primary key ascending to ensure consistent acquisition order

-- Transaction 1: id=1 then id=2
-- Transaction 2: id=1 then id=2  (same order - safe)
-- Never: Transaction 1 id=1 then id=2, Transaction 2 id=2 then id=1 - deadlock risk
```

## When to Use Pessimistic vs Optimistic Locking

Use pessimistic locking when:

- Conflicts are frequent and retries are expensive
- Business logic must complete atomically without retry
- Strict sequencing of updates is required (e.g., inventory deduction)

Use optimistic locking when conflicts are rare and scalability matters more than strict ordering.

## Summary

Pessimistic locking in MySQL uses `SELECT ... FOR UPDATE` to hold exclusive locks from the moment of reading, guaranteeing no concurrent modifications occur. It is simple to implement, eliminates retry logic, but reduces concurrency. Use SKIP LOCKED or NOWAIT options for non-blocking variants, and always acquire locks in a consistent order to prevent deadlocks.
