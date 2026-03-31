# How to Create a BEFORE UPDATE Trigger in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Trigger, BEFORE UPDATE, Validation, Audit

Description: Learn how to create a BEFORE UPDATE trigger in MySQL to validate or modify column values before an UPDATE statement commits changes to a table.

---

A `BEFORE UPDATE` trigger fires before MySQL applies an `UPDATE` statement to a row. Because it runs before the change is persisted, you can validate new values, override them, or raise an error to prevent the update entirely.

## Basic Syntax

```sql
CREATE TRIGGER trigger_name
BEFORE UPDATE ON table_name
FOR EACH ROW
BEGIN
    -- trigger body using NEW and OLD
END;
```

Inside a `BEFORE UPDATE` trigger:
- `OLD.column` holds the value before the update
- `NEW.column` holds the proposed new value
- You can assign to `NEW.column` to change the value that will be written

## Example 1 - Preventing Price Reduction Below a Minimum

```sql
DELIMITER //

CREATE TRIGGER trg_products_before_update
BEFORE UPDATE ON products
FOR EACH ROW
BEGIN
    IF NEW.price < 1.00 THEN
        SIGNAL SQLSTATE '45000'
            SET MESSAGE_TEXT = 'Price cannot be less than 1.00';
    END IF;
END //

DELIMITER ;
```

Test:

```sql
UPDATE products SET price = 0.50 WHERE product_id = 1;
-- ERROR 1644 (45000): Price cannot be less than 1.00
```

## Example 2 - Auto-stamping an Updated Timestamp

Force a column to always reflect the actual modification time regardless of what the application sends:

```sql
DELIMITER //

CREATE TRIGGER trg_users_stamp_updated
BEFORE UPDATE ON users
FOR EACH ROW
BEGIN
    SET NEW.updated_at = NOW();
END //

DELIMITER ;
```

This is useful when your ORM or application code may forget to set `updated_at`.

## Example 3 - Keeping an Audit Trail

Record every price change alongside the old and new values:

```sql
CREATE TABLE product_price_history (
    history_id  INT AUTO_INCREMENT PRIMARY KEY,
    product_id  INT,
    old_price   DECIMAL(10,2),
    new_price   DECIMAL(10,2),
    changed_by  VARCHAR(100),
    changed_at  DATETIME
);

DELIMITER //

CREATE TRIGGER trg_products_price_history
BEFORE UPDATE ON products
FOR EACH ROW
BEGIN
    IF OLD.price <> NEW.price THEN
        INSERT INTO product_price_history
            (product_id, old_price, new_price, changed_by, changed_at)
        VALUES
            (OLD.product_id, OLD.price, NEW.price, USER(), NOW());
    END IF;
END //

DELIMITER ;
```

The `IF OLD.price <> NEW.price` guard avoids inserting a history row when no price change occurred.

## Viewing and Managing the Trigger

```sql
-- List triggers on a table
SHOW TRIGGERS LIKE 'products';

-- View full trigger definition
SHOW CREATE TRIGGER trg_products_before_update;

-- Drop a trigger
DROP TRIGGER IF EXISTS trg_products_before_update;
```

## Summary

`BEFORE UPDATE` triggers are best suited for input validation, automatic column stamping, and conditional audit logging. Assigning to `NEW.column` inside the trigger body lets you override what gets written to the table, giving you fine-grained control over every update operation.
