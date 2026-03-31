# How to Use SAVEPOINT in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Savepoint, Transaction, InnoDB, Database Management

Description: Learn how to use SAVEPOINTs in MySQL to create partial rollback points within a transaction, enabling fine-grained error recovery.

---

## Overview

A `SAVEPOINT` is a named marker within a transaction that lets you roll back part of a transaction without aborting the entire thing. When something goes wrong mid-transaction, instead of rolling back all work done so far, you can roll back only to the most recent (or any named) savepoint and retry the failed operation.

SAVEPOINTs are supported by InnoDB tables and follow the SQL standard.

## Basic Syntax

```sql
SAVEPOINT savepoint_name;           -- Create a savepoint
ROLLBACK TO SAVEPOINT savepoint_name;  -- Roll back to a savepoint
RELEASE SAVEPOINT savepoint_name;   -- Remove a savepoint
```

## Simple Example

```sql
START TRANSACTION;

INSERT INTO orders (user_id, total, status) VALUES (1, 100.00, 'pending');
SAVEPOINT after_order;

INSERT INTO order_items (order_id, product_id, quantity) VALUES (LAST_INSERT_ID(), 5, 2);
SAVEPOINT after_items;

-- Something goes wrong with payment
INSERT INTO payments (order_id, amount, status) VALUES (LAST_INSERT_ID(), 100.00, 'invalid_method');

-- Roll back only the failed payment, keep order and items
ROLLBACK TO SAVEPOINT after_items;

-- Retry with correct payment method
INSERT INTO payments (order_id, amount, status) VALUES (10001, 100.00, 'credit_card');

COMMIT;
```

## Multiple Savepoints

You can create multiple savepoints within a single transaction:

```sql
START TRANSACTION;

INSERT INTO audit_log (action, user_id) VALUES ('batch_start', 42);
SAVEPOINT sp1;

UPDATE inventory SET quantity = quantity - 10 WHERE product_id = 101;
SAVEPOINT sp2;

UPDATE inventory SET quantity = quantity - 5 WHERE product_id = 202;
SAVEPOINT sp3;

-- Check if quantity went negative
SELECT quantity FROM inventory WHERE product_id = 202;

-- If quantity < 0, roll back only the last update
ROLLBACK TO SAVEPOINT sp2;

-- sp1 and sp2 still exist, sp3 was released by the rollback
COMMIT;
```

## Savepoints in Application Code

In a Node.js application with complex business logic:

```javascript
const conn = await pool.getConnection();

try {
  await conn.beginTransaction();
  
  // Step 1: Create the order
  const [orderResult] = await conn.execute(
    'INSERT INTO orders (user_id, total) VALUES (?, ?)',
    [userId, total]
  );
  const orderId = orderResult.insertId;
  
  await conn.execute('SAVEPOINT after_order');
  
  // Step 2: Reserve inventory
  try {
    for (const item of items) {
      const [rows] = await conn.execute(
        'SELECT quantity FROM inventory WHERE product_id = ? FOR UPDATE',
        [item.productId]
      );
      if (rows[0].quantity < item.qty) {
        throw new Error(`Insufficient stock for product ${item.productId}`);
      }
      await conn.execute(
        'UPDATE inventory SET quantity = quantity - ? WHERE product_id = ?',
        [item.qty, item.productId]
      );
    }
    await conn.execute('SAVEPOINT after_inventory');
  } catch (inventoryError) {
    await conn.execute('ROLLBACK TO SAVEPOINT after_order');
    // Order exists but inventory not reserved - handle accordingly
    throw inventoryError;
  }
  
  await conn.commit();
} catch (err) {
  await conn.rollback();
  throw err;
} finally {
  conn.release();
}
```

## Savepoint Naming Rules

Savepoint names follow identifier rules in MySQL:

```sql
-- Valid savepoint names
SAVEPOINT sp1;
SAVEPOINT after_insert;
SAVEPOINT `my savepoint`;  -- backtick-quoted for names with spaces

-- Case-insensitive in MySQL
SAVEPOINT MySP;
ROLLBACK TO SAVEPOINT mysp;  -- works
```

## What Happens to Savepoints After ROLLBACK TO

When you `ROLLBACK TO SAVEPOINT sp2`, savepoints created after `sp2` (like `sp3`) are automatically removed. `sp1` and `sp2` remain valid.

```sql
START TRANSACTION;
SAVEPOINT sp1;
SAVEPOINT sp2;
SAVEPOINT sp3;

ROLLBACK TO SAVEPOINT sp2;
-- sp3 is now gone; sp1 and sp2 still exist

ROLLBACK TO SAVEPOINT sp1;
-- sp2 is now gone; sp1 still exists

COMMIT;
```

## RELEASE SAVEPOINT

To explicitly release a savepoint (free its memory without rolling back):

```sql
START TRANSACTION;

INSERT INTO logs (message) VALUES ('operation started');
SAVEPOINT sp1;

-- Work succeeds, release the savepoint
RELEASE SAVEPOINT sp1;

COMMIT;
```

Savepoints are automatically released when a transaction is committed or fully rolled back.

## Summary

SAVEPOINTs in MySQL allow partial transaction rollbacks, enabling sophisticated error recovery patterns without losing all work done in a transaction. Create them with `SAVEPOINT name`, roll back to them with `ROLLBACK TO SAVEPOINT name`, and release them with `RELEASE SAVEPOINT name`. They are especially useful in complex application workflows where different parts of a transaction can fail independently and need selective retry logic.
