# How to Call One Stored Procedure from Another in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Stored Procedure, Modular Design, SQL, Programming

Description: Learn how to call one MySQL stored procedure from another to build modular, reusable database logic and simplify complex business workflows.

---

MySQL allows stored procedures to call other stored procedures using the `CALL` statement, just as you would call them from an application. This enables you to decompose complex logic into smaller, reusable units and build a library of database-level functions.

## Basic Nested Procedure Call

```sql
DELIMITER //

-- Helper procedure: validate customer exists and is active
CREATE PROCEDURE validate_customer(
    IN p_customer_id INT,
    OUT p_valid BOOLEAN
)
BEGIN
    DECLARE v_count INT;

    SELECT COUNT(*) INTO v_count
    FROM customers
    WHERE id = p_customer_id AND status = 'active';

    SET p_valid = (v_count > 0);
END //

-- Helper procedure: calculate applicable discount
CREATE PROCEDURE get_discount(
    IN p_customer_id INT,
    IN p_order_total DECIMAL(10,2),
    OUT p_discount_pct DECIMAL(5,2)
)
BEGIN
    DECLARE v_tier VARCHAR(20);

    SELECT tier INTO v_tier FROM customers WHERE id = p_customer_id;

    SET p_discount_pct = CASE v_tier
        WHEN 'gold' THEN 15.00
        WHEN 'silver' THEN 10.00
        WHEN 'bronze' THEN 5.00
        ELSE 0.00
    END;
END //

-- Main procedure calls both helpers
CREATE PROCEDURE create_order(
    IN p_customer_id INT,
    IN p_total DECIMAL(10,2),
    OUT p_order_id INT
)
BEGIN
    DECLARE v_valid BOOLEAN DEFAULT FALSE;
    DECLARE v_discount DECIMAL(5,2) DEFAULT 0;
    DECLARE v_final_total DECIMAL(10,2);

    -- Call helper to validate customer
    CALL validate_customer(p_customer_id, v_valid);

    IF NOT v_valid THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'Invalid or inactive customer';
    END IF;

    -- Call helper to get discount
    CALL get_discount(p_customer_id, p_total, v_discount);

    SET v_final_total = p_total * (1 - v_discount / 100);

    INSERT INTO orders (customer_id, subtotal, discount_pct, total, created_at)
    VALUES (p_customer_id, p_total, v_discount, v_final_total, NOW());

    SET p_order_id = LAST_INSERT_ID();
END //

DELIMITER ;
```

## Passing OUT Parameters Between Procedures

OUT and INOUT parameters work across nested calls:

```sql
DELIMITER //

CREATE PROCEDURE log_event(
    IN p_entity_type VARCHAR(50),
    IN p_entity_id INT,
    IN p_action VARCHAR(50),
    OUT p_log_id INT
)
BEGIN
    INSERT INTO audit_log (entity_type, entity_id, action, created_at)
    VALUES (p_entity_type, p_entity_id, p_action, NOW());

    SET p_log_id = LAST_INSERT_ID();
END //

CREATE PROCEDURE deactivate_customer(IN p_customer_id INT)
BEGIN
    DECLARE v_log_id INT;

    UPDATE customers SET status = 'inactive', updated_at = NOW()
    WHERE id = p_customer_id;

    -- Call audit logging procedure
    CALL log_event('customer', p_customer_id, 'deactivated', v_log_id);

    SELECT v_log_id AS audit_log_id,
           CONCAT('Customer ', p_customer_id, ' deactivated') AS message;
END //

DELIMITER ;
```

## Building a Pipeline of Procedures

For ETL-style workflows, chain procedures in sequence:

```sql
DELIMITER //

CREATE PROCEDURE extract_raw_data(IN p_batch_date DATE)
BEGIN
    INSERT INTO staging_orders
    SELECT * FROM raw_data
    WHERE DATE(created_at) = p_batch_date;
END //

CREATE PROCEDURE transform_staging_data()
BEGIN
    UPDATE staging_orders
    SET total = subtotal + tax + shipping
    WHERE total IS NULL;

    DELETE FROM staging_orders
    WHERE customer_id IS NULL OR subtotal <= 0;
END //

CREATE PROCEDURE load_processed_data()
BEGIN
    INSERT INTO orders_fact
    SELECT customer_id, total, DATE(created_at) AS order_date
    FROM staging_orders;

    TRUNCATE TABLE staging_orders;
END //

-- Orchestrator procedure calls the pipeline
CREATE PROCEDURE run_etl_pipeline(IN p_batch_date DATE)
BEGIN
    CALL extract_raw_data(p_batch_date);
    CALL transform_staging_data();
    CALL load_processed_data();

    SELECT CONCAT('ETL complete for ', p_batch_date) AS status;
END //

DELIMITER ;
```

## Error Propagation Across Nested Calls

Unhandled errors in a nested procedure propagate up to the caller:

```sql
DELIMITER //

CREATE PROCEDURE inner_proc()
BEGIN
    SIGNAL SQLSTATE '45000'
    SET MESSAGE_TEXT = 'Error from inner_proc';
END //

CREATE PROCEDURE outer_proc()
BEGIN
    DECLARE EXIT HANDLER FOR SQLSTATE '45000'
    BEGIN
        SELECT 'Caught error from inner_proc' AS error_message;
    END;

    CALL inner_proc();  -- Error propagates here
    SELECT 'This line is never reached' AS msg;
END //

DELIMITER ;

CALL outer_proc();
-- Returns: Caught error from inner_proc
```

## Recursion Limits

MySQL supports recursive stored procedure calls up to the depth set by `max_sp_recursion_depth` (default 0 - no recursion):

```sql
-- Enable recursion (use with caution)
SET SESSION max_sp_recursion_depth = 10;
```

## Summary

MySQL stored procedures call other procedures with `CALL proc_name(args)`. This enables modular design where helper procedures handle validation, logging, and reusable calculations. OUT parameters pass values between nested calls. Errors in inner procedures propagate up unless caught by a handler in the outer procedure. For complex workflows, build orchestrator procedures that coordinate a pipeline of specialized sub-procedures.
