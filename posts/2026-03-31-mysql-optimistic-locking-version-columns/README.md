# How to Implement Optimistic Locking with Version Columns in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Concurrency, Transaction, Lock, Pattern

Description: Learn how to implement optimistic locking in MySQL using version columns to prevent lost updates without holding database locks.

---

## What Is Optimistic Locking?

Optimistic locking assumes that concurrent conflicts are rare. Instead of holding a row lock while a user reads and edits data, you record the row's version number when reading, then check at update time that the version has not changed. If another process updated the row between your read and write, the update affects zero rows and you handle the conflict in application code.

This is in contrast to pessimistic locking, which uses `SELECT ... FOR UPDATE` to hold a row lock throughout the entire user interaction.

## Schema Design

```sql
CREATE TABLE products (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    price DECIMAL(10,2) NOT NULL,
    stock_quantity INT NOT NULL DEFAULT 0,
    version INT NOT NULL DEFAULT 0,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

INSERT INTO products (name, price, stock_quantity, version)
VALUES ('Widget', 9.99, 100, 0);
```

## The Optimistic Lock Pattern

```sql
-- Step 1: Read the row and its version
SELECT id, name, price, stock_quantity, version
FROM products
WHERE id = 1;
-- Returns: id=1, price=9.99, stock_quantity=100, version=0

-- Step 2: User makes changes (in application, no DB lock held)

-- Step 3: Update only if version matches
UPDATE products
SET price = 12.99,
    version = version + 1
WHERE id = 1
  AND version = 0;  -- The version we read in Step 1

-- Check rows affected
-- 1 row affected = success, our update won
-- 0 rows affected = conflict, another process updated first
```

## Application-Level Implementation

```python
import mysql.connector

def update_product_price(product_id, new_price):
    conn = mysql.connector.connect(
        host='localhost', database='myapp',
        user='app_user', password='secret'
    )
    cursor = conn.cursor(dictionary=True)
    max_retries = 3

    for attempt in range(max_retries):
        # Read current state including version
        cursor.execute(
            'SELECT id, price, version FROM products WHERE id = %s',
            (product_id,)
        )
        product = cursor.fetchone()
        current_version = product['version']

        # Attempt the optimistic update
        cursor.execute("""
            UPDATE products
            SET price = %s, version = version + 1
            WHERE id = %s AND version = %s
        """, (new_price, product_id, current_version))
        conn.commit()

        if cursor.rowcount == 1:
            print(f"Update succeeded on attempt {attempt + 1}")
            return True
        else:
            print(f"Conflict detected on attempt {attempt + 1}, retrying...")

    print("Update failed after max retries")
    return False
```

## Using Timestamps Instead of Integers

An alternative uses the `updated_at` timestamp as the version token:

```sql
-- Read
SELECT id, name, price, updated_at FROM products WHERE id = 1;
-- Returns updated_at = '2025-12-01 10:00:00'

-- Update with timestamp version check
UPDATE products
SET price = 12.99
WHERE id = 1
  AND updated_at = '2025-12-01 10:00:00';
```

Integer version columns are more reliable than timestamps because timestamp precision can cause missed conflicts on fast concurrent updates.

## Detecting and Handling Conflicts

```sql
-- After UPDATE, check affected rows in a stored procedure
DELIMITER $$
CREATE PROCEDURE update_product(
    IN p_id INT,
    IN p_new_price DECIMAL(10,2),
    IN p_expected_version INT,
    OUT p_success BOOLEAN
)
BEGIN
    UPDATE products
    SET price = p_new_price,
        version = version + 1
    WHERE id = p_id
      AND version = p_expected_version;

    SET p_success = (ROW_COUNT() = 1);
END$$
DELIMITER ;

-- Call the procedure
CALL update_product(1, 12.99, 0, @success);
SELECT @success;
```

## When to Use Optimistic vs Pessimistic Locking

Use optimistic locking when:
- Read-to-write time is long (user interaction involved)
- Conflicts are infrequent
- You want to avoid lock wait timeouts

Use pessimistic locking (`SELECT ... FOR UPDATE`) when:
- Conflicts are frequent
- You cannot tolerate retries
- The critical section is short and bounded

## Summary

Optimistic locking with version columns in MySQL prevents lost updates without holding database row locks during user interactions. The pattern is simple: read a row's version, make changes in application memory, then update only if the version matches. If not, retry or notify the user of a conflict. This approach scales well for long read-modify-write cycles common in web applications.
