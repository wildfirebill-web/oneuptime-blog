# How to Implement Data Validation with MySQL Triggers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Trigger, Validation, BEFORE INSERT, SIGNAL

Description: Learn how to enforce data validation rules in MySQL using BEFORE INSERT and BEFORE UPDATE triggers with SIGNAL to reject invalid data.

---

MySQL CHECK constraints (available since 8.0.16) handle simple column-level rules, but triggers let you implement complex, cross-table validation logic that runs automatically before every INSERT or UPDATE. When validation fails, a `SIGNAL` inside the trigger aborts the statement and returns a descriptive error to the caller.

## Why Use Triggers for Validation

- Enforce rules that span multiple columns or tables
- Apply validation for all clients, including direct SQL connections, not just the application
- Return a meaningful error message rather than a generic constraint violation

## Basic Pattern

```sql
DELIMITER //

CREATE TRIGGER trg_table_before_insert
BEFORE INSERT ON table_name
FOR EACH ROW
BEGIN
    IF <invalid condition> THEN
        SIGNAL SQLSTATE '45000'
            SET MESSAGE_TEXT = 'Descriptive error message';
    END IF;
END //

DELIMITER ;
```

`SQLSTATE '45000'` is the generic "unhandled user-defined exception" state. You can also use `'22007'` (invalid datetime value) or other standard codes when appropriate.

## Example 1 - Validate Age Range

```sql
DELIMITER //

CREATE TRIGGER trg_users_validate_age
BEFORE INSERT ON users
FOR EACH ROW
BEGIN
    IF NEW.age < 18 OR NEW.age > 120 THEN
        SIGNAL SQLSTATE '45000'
            SET MESSAGE_TEXT = 'Age must be between 18 and 120';
    END IF;
END //

DELIMITER ;
```

Test:

```sql
INSERT INTO users (name, age) VALUES ('Alice', 15);
-- ERROR 1644 (45000): Age must be between 18 and 120
```

## Example 2 - Cross-Column Validation

Ensure a discount cannot exceed the price:

```sql
DELIMITER //

CREATE TRIGGER trg_products_validate_discount
BEFORE INSERT ON products
FOR EACH ROW
BEGIN
    IF NEW.discount > NEW.price THEN
        SIGNAL SQLSTATE '45000'
            SET MESSAGE_TEXT = 'Discount cannot exceed the product price';
    END IF;
END //

-- Apply the same check on updates
CREATE TRIGGER trg_products_update_validate_discount
BEFORE UPDATE ON products
FOR EACH ROW
BEGIN
    IF NEW.discount > NEW.price THEN
        SIGNAL SQLSTATE '45000'
            SET MESSAGE_TEXT = 'Discount cannot exceed the product price';
    END IF;
END //

DELIMITER ;
```

## Example 3 - Cross-Table Validation

Prevent inserting an order for a product that is out of stock:

```sql
DELIMITER //

CREATE TRIGGER trg_order_items_check_stock
BEFORE INSERT ON order_items
FOR EACH ROW
BEGIN
    DECLARE v_stock INT DEFAULT 0;

    SELECT stock_qty INTO v_stock
    FROM inventory
    WHERE product_id = NEW.product_id;

    IF v_stock < NEW.quantity THEN
        SIGNAL SQLSTATE '45000'
            SET MESSAGE_TEXT = 'Insufficient stock for this product';
    END IF;
END //

DELIMITER ;
```

## Normalizing Data in the Same Trigger

You can combine validation and normalization - modify `NEW` if valid, reject if not:

```sql
DELIMITER //

CREATE TRIGGER trg_contacts_normalize_phone
BEFORE INSERT ON contacts
FOR EACH ROW
BEGIN
    -- Remove non-numeric characters
    SET NEW.phone = REGEXP_REPLACE(NEW.phone, '[^0-9]', '');

    IF LENGTH(NEW.phone) <> 10 THEN
        SIGNAL SQLSTATE '45000'
            SET MESSAGE_TEXT = 'Phone number must have exactly 10 digits';
    END IF;
END //

DELIMITER ;
```

## Summary

Use `BEFORE INSERT` and `BEFORE UPDATE` triggers with `SIGNAL SQLSTATE '45000'` to enforce complex validation rules at the database layer. This ensures data integrity regardless of the client, covers cross-column and cross-table rules that CHECK constraints cannot express, and returns readable error messages to applications.
