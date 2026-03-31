# How to Create an AFTER DELETE Trigger in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Trigger, AFTER DELETE, Audit, Cascade

Description: Learn how to create an AFTER DELETE trigger in MySQL to clean up related data, update aggregates, or audit row removals after a deletion succeeds.

---

An `AFTER DELETE` trigger fires after MySQL has successfully deleted a row. Because the deletion has already taken place, you cannot prevent it inside this trigger - that is the job of a `BEFORE DELETE` trigger. However, `AFTER DELETE` is the right place to cascade side effects: cleaning up related records, updating counters, or writing audit logs.

## Basic Syntax

```sql
CREATE TRIGGER trigger_name
AFTER DELETE ON table_name
FOR EACH ROW
BEGIN
    -- trigger body using OLD
END;
```

`OLD.column` holds the values of the deleted row. `NEW` does not exist in a delete trigger.

## Example 1 - Clean Up Related Records

When a user account is deleted, remove their related sessions and preferences:

```sql
DELIMITER //

CREATE TRIGGER trg_users_after_delete
AFTER DELETE ON users
FOR EACH ROW
BEGIN
    DELETE FROM user_sessions    WHERE user_id = OLD.user_id;
    DELETE FROM user_preferences WHERE user_id = OLD.user_id;
END //

DELIMITER ;
```

Note: if your schema uses `ON DELETE CASCADE` foreign keys, MySQL handles cascades automatically. Use a trigger only when the related tables do not have a foreign key relationship.

## Example 2 - Decrement a Counter

Maintain a category product count when a product is deleted:

```sql
DELIMITER //

CREATE TRIGGER trg_products_after_delete
AFTER DELETE ON products
FOR EACH ROW
BEGIN
    UPDATE categories
    SET product_count = product_count - 1
    WHERE category_id = OLD.category_id;
END //

DELIMITER ;
```

## Example 3 - Write a Deletion Audit Record

```sql
CREATE TABLE deletion_log (
    log_id     INT AUTO_INCREMENT PRIMARY KEY,
    table_name VARCHAR(64),
    record_id  INT,
    deleted_by VARCHAR(100),
    deleted_at DATETIME
);

DELIMITER //

CREATE TRIGGER trg_orders_after_delete
AFTER DELETE ON orders
FOR EACH ROW
BEGIN
    INSERT INTO deletion_log (table_name, record_id, deleted_by, deleted_at)
    VALUES ('orders', OLD.order_id, USER(), NOW());
END //

DELIMITER ;
```

Test:

```sql
DELETE FROM orders WHERE order_id = 10;
SELECT * FROM deletion_log;
```

## AFTER DELETE vs BEFORE DELETE

```text
Use BEFORE DELETE when you need to:   | Use AFTER DELETE when you need to:
--------------------------------------|------------------------------------
Prevent the deletion (SIGNAL)         | Clean up related rows
Archive the row to another table      | Update aggregate counters
Validate business rules               | Write an audit trail
```

## Viewing and Removing the Trigger

```sql
SHOW TRIGGERS LIKE 'orders';
SHOW CREATE TRIGGER trg_orders_after_delete;
DROP TRIGGER IF EXISTS trg_orders_after_delete;
```

## Summary

An `AFTER DELETE` trigger is ideal for cascading cleanups, counter updates, and audit logging once a row has been successfully removed. Access deleted column values with `OLD.column`, and keep the trigger body concise to avoid adding latency to every `DELETE` operation on the table.
