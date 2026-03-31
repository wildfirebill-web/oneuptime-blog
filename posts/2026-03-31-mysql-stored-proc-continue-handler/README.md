# How to Use DECLARE CONTINUE HANDLER in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Stored Procedure, Error Handling, Handler, SQL

Description: Learn how to use DECLARE CONTINUE HANDLER in MySQL stored procedures to catch errors and warnings while allowing execution to continue.

---

## What Is a CONTINUE HANDLER?

In MySQL stored procedures, a `DECLARE CONTINUE HANDLER` catches a specific condition (error, warning, or SQLSTATE) and runs a handler block. After the handler body executes, control returns to the statement immediately after the one that triggered the condition. This lets you log errors, set flags, and keep processing without aborting the procedure.

## Syntax

```sql
DECLARE CONTINUE HANDLER FOR condition_value
BEGIN
    -- handler actions
END;
```

`condition_value` can be:
- A named condition: `SQLWARNING`, `NOT FOUND`, `SQLEXCEPTION`
- A specific SQLSTATE: `SQLSTATE '02000'`
- A MySQL error number: `1062` (duplicate entry)

## Handling "No Rows Found" with NOT FOUND

The most common use of a CONTINUE handler is catching the `NOT FOUND` condition when a `SELECT INTO` returns no rows:

```sql
DELIMITER $$
CREATE PROCEDURE get_customer_email(IN p_id INT, OUT p_email VARCHAR(255))
BEGIN
    DECLARE v_done INT DEFAULT 0;

    DECLARE CONTINUE HANDLER FOR NOT FOUND
    BEGIN
        SET v_done = 1;
        SET p_email = NULL;
    END;

    SELECT email INTO p_email FROM customers WHERE id = p_id;

    IF v_done = 1 THEN
        SELECT 'Customer not found' AS message;
    ELSE
        SELECT CONCAT('Email: ', p_email) AS message;
    END IF;
END$$
DELIMITER ;

CALL get_customer_email(999, @email);
```

## Handling Duplicate Key Errors

```sql
DELIMITER $$
CREATE PROCEDURE safe_insert_tag(IN p_name VARCHAR(100), OUT p_result VARCHAR(50))
BEGIN
    DECLARE CONTINUE HANDLER FOR 1062
    BEGIN
        SET p_result = 'duplicate';
    END;

    SET p_result = 'inserted';
    INSERT INTO tags (name) VALUES (p_name);
END$$
DELIMITER ;

CALL safe_insert_tag('mysql', @res);
SELECT @res;
```

If the tag already exists, the duplicate key error is caught, `p_result` is set to `'duplicate'`, and the procedure returns normally.

## Using a CONTINUE Handler in a Cursor Loop

CONTINUE handlers are essential when iterating with cursors. The `NOT FOUND` condition signals that the cursor has no more rows:

```sql
DELIMITER $$
CREATE PROCEDURE process_all_orders()
BEGIN
    DECLARE v_order_id INT;
    DECLARE v_done     BOOLEAN DEFAULT FALSE;

    DECLARE order_cur CURSOR FOR
        SELECT id FROM orders WHERE processed = 0;

    DECLARE CONTINUE HANDLER FOR NOT FOUND
        SET v_done = TRUE;

    OPEN order_cur;

    read_loop: LOOP
        FETCH order_cur INTO v_order_id;
        IF v_done THEN
            LEAVE read_loop;
        END IF;
        -- process each order
        UPDATE orders SET processed = 1 WHERE id = v_order_id;
    END LOOP;

    CLOSE order_cur;
END$$
DELIMITER ;
```

## Handling All SQL Exceptions

```sql
DELIMITER $$
CREATE PROCEDURE batch_update_prices(IN p_factor DECIMAL(4,2))
BEGIN
    DECLARE v_error_count INT DEFAULT 0;

    DECLARE CONTINUE HANDLER FOR SQLEXCEPTION
    BEGIN
        SET v_error_count = v_error_count + 1;
    END;

    UPDATE products SET price = price * p_factor WHERE category = 'electronics';
    UPDATE products SET price = price * p_factor WHERE category = 'clothing';

    SELECT v_error_count AS errors_encountered;
END$$
DELIMITER ;
```

## CONTINUE vs EXIT Handler

| Handler Type | After Handler Runs |
|---|---|
| CONTINUE | Resumes execution after the triggering statement |
| EXIT | Exits the current BEGIN...END block |

Use `CONTINUE` when you want the procedure to keep running after catching a non-fatal condition. Use `EXIT` when the error is unrecoverable and you want to stop the current block.

## Summary

`DECLARE CONTINUE HANDLER` allows MySQL stored procedures to catch errors and warnings and then resume execution. It is most commonly used with `NOT FOUND` in cursor loops and with specific error codes like `1062` for duplicate-key violations. Set a flag variable inside the handler to communicate the error state to the rest of the procedure.
