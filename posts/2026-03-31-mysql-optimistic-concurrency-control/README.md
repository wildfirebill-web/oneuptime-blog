# How to Implement Optimistic Concurrency Control in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Transaction, Concurrency

Description: Learn how to implement optimistic concurrency control in MySQL using version columns and conditional UPDATE statements to prevent lost update anomalies.

---

Optimistic concurrency control (OCC) assumes conflicts are rare and defers conflict detection to the moment of write. Instead of locking rows during reads, it uses a version number or timestamp to detect whether another transaction modified the row between the read and the write.

## How It Works

1. Read a row and capture its current version.
2. Perform application logic using the read data.
3. Update the row only if the version has not changed.
4. If the update affects zero rows, another transaction modified the row and you must retry.

## Schema Design

Add a `version` column to any table that needs OCC:

```sql
CREATE TABLE products (
  id       INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  name     VARCHAR(100) NOT NULL,
  stock    INT NOT NULL DEFAULT 0,
  price    DECIMAL(10, 2) NOT NULL,
  version  INT UNSIGNED NOT NULL DEFAULT 0
);
```

Alternatively, use a `TIMESTAMP` or `DATETIME` for the version:

```sql
ALTER TABLE products
ADD COLUMN updated_at DATETIME(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6)
  ON UPDATE CURRENT_TIMESTAMP(6);
```

## Read Phase

Read the row and capture the version:

```sql
SELECT id, stock, price, version
FROM products
WHERE id = 42;
```

Store `version = 5` in the application layer.

## Write Phase with Version Check

When updating, include the captured version in the `WHERE` clause:

```sql
UPDATE products
SET
  stock   = stock - 1,
  version = version + 1
WHERE id = 42
  AND version = 5;
```

Check the number of affected rows after the update:

```sql
SELECT ROW_COUNT();
```

If `ROW_COUNT()` returns 0, the version changed between the read and the write, indicating a conflict.

## Handling Conflicts in Application Code

```python
import mysql.connector

def decrement_stock(conn, product_id, quantity):
    cursor = conn.cursor(dictionary=True)
    max_retries = 3

    for attempt in range(max_retries):
        cursor.execute(
            "SELECT id, stock, version FROM products WHERE id = %s",
            (product_id,)
        )
        row = cursor.fetchone()

        if row['stock'] < quantity:
            raise ValueError("Insufficient stock")

        cursor.execute(
            "UPDATE products SET stock = stock - %s, version = version + 1 "
            "WHERE id = %s AND version = %s",
            (quantity, product_id, row['version'])
        )
        conn.commit()

        if cursor.rowcount == 1:
            return True  # success

    raise Exception("Too many conflicts, could not update")
```

## Timestamp-Based OCC

Using `updated_at` instead of a counter:

```sql
-- Read
SELECT id, stock, updated_at FROM products WHERE id = 42;

-- Write
UPDATE products
SET stock = stock - 1
WHERE id = 42
  AND updated_at = '2026-03-31 10:15:30.123456';
```

Timestamps have a small risk of false matches if two writes happen within the same microsecond on high-traffic systems, so integer version counters are more reliable.

## When to Use Optimistic vs Pessimistic Locking

Use OCC when conflicts are infrequent and the work between read and write is short. Use `SELECT ... FOR UPDATE` (pessimistic locking) when contention is high or when the application logic cannot safely retry, such as financial debits.

## Summary

Optimistic concurrency control in MySQL uses a version column and a conditional `UPDATE ... WHERE version = ?` pattern. If the update returns zero rows, a conflict occurred and the operation should retry. This approach scales better than row locks for low-contention workloads because readers never block writers.
