# How to Use DECLARE EXIT HANDLER in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Stored Procedure, Error Handling, Handler, SQL

Description: Learn how to use DECLARE EXIT HANDLER in MySQL stored procedures to catch errors, run cleanup code, and exit the current block gracefully.

---

## What Is an EXIT HANDLER?

A `DECLARE EXIT HANDLER` in MySQL catches a condition (error, warning, or SQLSTATE) and runs a handler block. After the handler block finishes, execution exits the `BEGIN...END` block in which the handler was declared - not the entire procedure unless the handler is in the outermost block. This is useful for rollback logic and error logging when an unrecoverable error occurs.

## Syntax

```sql
DECLARE EXIT HANDLER FOR condition_value
BEGIN
    -- cleanup or logging
END;
```

`condition_value` can be:
- `SQLEXCEPTION` - any SQL error
- `SQLWARNING` - any warning
- `NOT FOUND` - no rows found condition
- A SQLSTATE string: `SQLSTATE '23000'`
- A MySQL error number: `1062`

## Basic EXIT HANDLER Example

```sql
DELIMITER $$
CREATE PROCEDURE create_order(IN p_customer_id INT, IN p_total DECIMAL(12,2))
BEGIN
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
        SELECT 'Order creation failed' AS error_message;
    END;

    START TRANSACTION;

    INSERT INTO orders (customer_id, total, created_at)
    VALUES (p_customer_id, p_total, NOW());

    INSERT INTO order_audit (order_id, action, performed_at)
    VALUES (LAST_INSERT_ID(), 'created', NOW());

    COMMIT;
END$$
DELIMITER ;
```

If any statement inside the transaction fails, the EXIT handler fires, rolls back the transaction, and returns an error message to the caller.

## EXIT HANDLER with Error Logging

```sql
DELIMITER $$
CREATE PROCEDURE update_inventory(IN p_product_id INT, IN p_delta INT)
BEGIN
    DECLARE v_sqlstate VARCHAR(5)  DEFAULT '00000';
    DECLARE v_msg      TEXT;
    DECLARE v_errno    INT;

    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        GET DIAGNOSTICS CONDITION 1
            v_sqlstate = RETURNED_SQLSTATE,
            v_errno    = MYSQL_ERRNO,
            v_msg      = MESSAGE_TEXT;

        INSERT INTO procedure_errors (procedure_name, sqlstate, errno, message, error_time)
        VALUES ('update_inventory', v_sqlstate, v_errno, v_msg, NOW());

        ROLLBACK;
        RESIGNAL;
    END;

    START TRANSACTION;
    UPDATE products SET stock = stock + p_delta WHERE id = p_product_id;
    COMMIT;
END$$
DELIMITER ;
```

`GET DIAGNOSTICS` captures structured error details so you can log them before exiting.

## Handling a Specific Error Code

```sql
DELIMITER $$
CREATE PROCEDURE add_unique_tag(IN p_tag VARCHAR(100))
BEGIN
    DECLARE EXIT HANDLER FOR 1062
    BEGIN
        SELECT CONCAT('Tag already exists: ', p_tag) AS message;
    END;

    INSERT INTO tags (name) VALUES (p_tag);
    SELECT 'Tag created successfully' AS message;
END$$
DELIMITER ;
```

## EXIT vs CONTINUE Handler

| Behavior | CONTINUE HANDLER | EXIT HANDLER |
|---|---|---|
| After handler runs | Resumes after triggering statement | Exits the BEGIN...END block |
| Transaction rollback | Must be done explicitly | Can be done before exit |
| Typical use | Non-fatal errors, cursor loops | Fatal errors, cleanup before aborting |

## Nested Blocks and Scope

An EXIT handler only exits its own `BEGIN...END` block. If the handler is in an inner block, execution continues in the outer block after the inner block exits:

```sql
DELIMITER $$
CREATE PROCEDURE example()
BEGIN
    -- outer block continues after inner block exits
    BEGIN
        DECLARE EXIT HANDLER FOR SQLEXCEPTION
            SELECT 'Inner block error handled' AS msg;

        INSERT INTO nonexistent_table VALUES (1);  -- triggers handler
    END;

    SELECT 'Outer block still running' AS msg;  -- this executes
END$$
DELIMITER ;
```

## Summary

`DECLARE EXIT HANDLER` catches SQL errors and exits the current `BEGIN...END` block after running cleanup actions like `ROLLBACK` and error logging. Use it for unrecoverable errors where you want to stop the current block, log context, and optionally re-raise the condition with `RESIGNAL`. Pair it with `GET DIAGNOSTICS` to capture structured error metadata before exiting.
