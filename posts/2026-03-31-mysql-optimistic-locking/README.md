# How to Implement Optimistic Locking in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Transaction, Locking, Concurrency, InnoDB

Description: Learn how to implement optimistic locking in MySQL using version columns to detect concurrent updates without holding database locks.

---

## What Is Optimistic Locking?

Optimistic locking is a concurrency control pattern where multiple transactions can read the same row simultaneously, but a conflict check is performed at write time. Instead of acquiring locks when reading, you include a version or timestamp column and verify it has not changed before committing an update. If it has changed, the update is retried or rejected.

## Schema Design for Optimistic Locking

Add a `version` column to the table:

```sql
CREATE TABLE products (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100) NOT NULL,
    price DECIMAL(10, 2) NOT NULL,
    stock INT NOT NULL,
    version INT NOT NULL DEFAULT 1
);

INSERT INTO products (name, price, stock, version)
VALUES ('Widget', 29.99, 100, 1);
```

## The Read-Modify-Write Pattern

```sql
-- Step 1: Read the row and remember the version
SELECT id, name, price, stock, version
FROM products
WHERE id = 1;
-- Returns: id=1, price=29.99, stock=100, version=1

-- Step 2: Modify data in application logic

-- Step 3: Update only if version matches
UPDATE products
SET price = 34.99,
    stock = 99,
    version = version + 1
WHERE id = 1
  AND version = 1;
-- Check ROW_COUNT() - if 0, a conflict occurred

SELECT ROW_COUNT();
-- Returns 1: success, 0: conflict detected
```

If another transaction modified the row between the read and write, the `version` no longer matches and ROW_COUNT() returns 0 - the conflict must be handled.

## Using a Timestamp Column Instead of Version

An alternative uses a `updated_at` timestamp:

```sql
ALTER TABLE products
    ADD COLUMN updated_at TIMESTAMP NOT NULL
        DEFAULT CURRENT_TIMESTAMP
        ON UPDATE CURRENT_TIMESTAMP;

-- Read the timestamp
SELECT id, price, updated_at FROM products WHERE id = 1;

-- Update only if timestamp matches
UPDATE products
SET price = 34.99
WHERE id = 1
  AND updated_at = '2026-03-31 10:00:00';

SELECT ROW_COUNT();
```

## Application-Level Conflict Handling

In application code (Python example):

```python
import mysql.connector

def update_product_price(product_id, new_price, retries=3):
    conn = mysql.connector.connect(host='localhost', database='shop',
                                   user='app', password='secret')
    cursor = conn.cursor(dictionary=True)

    for attempt in range(retries):
        cursor.execute(
            "SELECT id, price, version FROM products WHERE id = %s",
            (product_id,)
        )
        row = cursor.fetchone()
        current_version = row['version']

        cursor.execute(
            """UPDATE products SET price = %s, version = version + 1
               WHERE id = %s AND version = %s""",
            (new_price, product_id, current_version)
        )
        conn.commit()

        if cursor.rowcount == 1:
            print("Update successful")
            return True
        # Conflict: retry

    print("Failed after retries")
    return False
```

## When to Use Optimistic Locking

Optimistic locking is best when:

- Conflicts are rare (most updates succeed without contention)
- Long read operations are needed before an update
- Avoiding database-held locks improves scalability
- Multiple application servers are updating the same rows

## When to Use Pessimistic Locking Instead

Use `SELECT ... FOR UPDATE` (pessimistic locking) when:

- Conflicts are frequent and retry overhead is significant
- Strict ordering of updates is required
- The read-modify-write cycle must be atomic with no retry logic

## Summary

Optimistic locking in MySQL is implemented by adding a version or timestamp column and verifying it at update time with a conditional WHERE clause. If the version has changed, the update affects zero rows, signaling a conflict. This pattern eliminates database lock holding during the read phase, improving scalability for workloads with infrequent conflicts.
