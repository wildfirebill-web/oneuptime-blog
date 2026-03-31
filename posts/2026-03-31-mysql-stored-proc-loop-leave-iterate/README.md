# How to Use LOOP with LEAVE and ITERATE in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Stored Procedure, Loop, Control Flow, SQL

Description: Learn how to use MySQL LOOP with LEAVE and ITERATE statements in stored procedures for precise loop control and early exit patterns.

---

MySQL's `LOOP` statement creates an unconditional loop that executes indefinitely until a `LEAVE` statement exits it. Combined with `ITERATE` (which acts like `continue` in other languages), `LOOP` provides full control over iteration flow in stored procedures.

## Basic LOOP with LEAVE

A bare `LOOP` runs forever - you must use `LEAVE` to exit it. Label the loop to identify it:

```sql
DELIMITER //

CREATE PROCEDURE count_down(IN p_start INT)
BEGIN
    DECLARE v_counter INT;
    SET v_counter = p_start;

    count_loop: LOOP
        IF v_counter <= 0 THEN
            LEAVE count_loop;
        END IF;

        INSERT INTO countdown_log (value, logged_at)
        VALUES (v_counter, NOW());

        SET v_counter = v_counter - 1;
    END LOOP count_loop;

    SELECT CONCAT('Counted down from ', p_start) AS result;
END //

DELIMITER ;

CALL count_down(5);
```

## Using ITERATE to Skip Iterations

`ITERATE` restarts the current loop from the top, similar to `continue` in C, Java, or Python:

```sql
DELIMITER //

CREATE PROCEDURE process_pending_orders()
BEGIN
    DECLARE v_done INT DEFAULT FALSE;
    DECLARE v_order_id INT;
    DECLARE v_customer_id INT;
    DECLARE v_status VARCHAR(20);
    DECLARE v_processed INT DEFAULT 0;
    DECLARE v_skipped INT DEFAULT 0;

    DECLARE order_cursor CURSOR FOR
        SELECT id, customer_id, status
        FROM orders
        WHERE status IN ('pending', 'on_hold', 'ready')
        ORDER BY created_at;

    DECLARE CONTINUE HANDLER FOR NOT FOUND SET v_done = TRUE;

    OPEN order_cursor;

    process_loop: LOOP
        FETCH order_cursor INTO v_order_id, v_customer_id, v_status;

        IF v_done THEN
            LEAVE process_loop;
        END IF;

        -- Skip on_hold orders
        IF v_status = 'on_hold' THEN
            SET v_skipped = v_skipped + 1;
            ITERATE process_loop;
        END IF;

        -- Process pending and ready orders
        UPDATE orders SET processed_at = NOW(), status = 'processing'
        WHERE id = v_order_id;

        INSERT INTO order_events (order_id, event_type)
        VALUES (v_order_id, 'processing_started');

        SET v_processed = v_processed + 1;
    END LOOP process_loop;

    CLOSE order_cursor;

    SELECT v_processed AS orders_processed,
           v_skipped AS orders_skipped;
END //

DELIMITER ;
```

## Nested Loops with Multiple Labels

Labels allow `LEAVE` and `ITERATE` to target a specific loop in nested structures:

```sql
DELIMITER //

CREATE PROCEDURE batch_matrix_process()
BEGIN
    DECLARE v_row INT DEFAULT 1;
    DECLARE v_col INT DEFAULT 1;
    DECLARE v_max_rows INT DEFAULT 10;
    DECLARE v_max_cols INT DEFAULT 10;
    DECLARE v_value INT;

    outer_loop: LOOP
        IF v_row > v_max_rows THEN
            LEAVE outer_loop;
        END IF;

        SET v_col = 1;

        inner_loop: LOOP
            IF v_col > v_max_cols THEN
                LEAVE inner_loop;
            END IF;

            SET v_value = v_row * v_col;

            -- Skip zero results (none here, but demonstrates the pattern)
            IF v_value = 0 THEN
                SET v_col = v_col + 1;
                ITERATE inner_loop;
            END IF;

            INSERT INTO matrix_data (row_num, col_num, val)
            VALUES (v_row, v_col, v_value);

            SET v_col = v_col + 1;
        END LOOP inner_loop;

        SET v_row = v_row + 1;
    END LOOP outer_loop;
END //

DELIMITER ;
```

## LOOP vs. WHILE vs. REPEAT

All three loop types achieve similar goals with different syntax:

```sql
-- LOOP: exit condition inside with LEAVE
my_loop: LOOP
    IF v_i > 10 THEN LEAVE my_loop; END IF;
    SET v_i = v_i + 1;
END LOOP;

-- WHILE: pre-condition check
WHILE v_i <= 10 DO
    SET v_i = v_i + 1;
END WHILE;

-- REPEAT: post-condition check (always executes at least once)
REPEAT
    SET v_i = v_i + 1;
UNTIL v_i > 10 END REPEAT;
```

Use `LOOP` with `LEAVE` when the exit condition is complex or must be checked at multiple points within the loop body. Use `WHILE` for simple pre-condition loops.

## Practical Use: Batch Processing with Limit

```sql
DELIMITER //

CREATE PROCEDURE archive_old_records(IN p_batch_size INT)
BEGIN
    DECLARE v_affected INT DEFAULT 1;

    archive_loop: LOOP
        IF v_affected = 0 THEN
            LEAVE archive_loop;
        END IF;

        INSERT INTO orders_archive
        SELECT * FROM orders
        WHERE created_at < DATE_SUB(NOW(), INTERVAL 2 YEAR)
        LIMIT p_batch_size;

        SET v_affected = ROW_COUNT();

        DELETE FROM orders
        WHERE created_at < DATE_SUB(NOW(), INTERVAL 2 YEAR)
        LIMIT p_batch_size;

        DO SLEEP(0.1);
    END LOOP archive_loop;
END //

DELIMITER ;
```

## Summary

MySQL's `LOOP` statement creates an infinite loop that continues until a `LEAVE label` statement exits it. Use `ITERATE label` to skip to the next iteration without executing the remaining loop body. Label your loops to control nested loop flow precisely. Choose `LOOP` when exit conditions are complex or must be checked at multiple points; use `WHILE` for simpler pre-condition loops.
