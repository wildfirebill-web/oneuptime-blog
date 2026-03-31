# How to Use BEGIN...END Compound Statements in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, BEGIN...END, Stored Procedure, Compound Statement

Description: Learn how to use BEGIN...END compound statements in MySQL stored programs to group multiple statements, declare local variables, and define condition handlers.

---

## What Is a BEGIN...END Compound Statement

A `BEGIN...END` compound statement groups multiple SQL statements into a single logical block. It is the fundamental building block of MySQL stored procedures, functions, triggers, and events. Inside a `BEGIN...END` block you can declare local variables, define condition handlers, and nest additional compound statements.

## Basic Structure

```sql
DELIMITER $$
CREATE PROCEDURE example_proc()
BEGIN
  -- Variable declarations must come first
  DECLARE v_count   INT     DEFAULT 0;
  DECLARE v_message VARCHAR(200);

  -- Condition handlers after declarations
  DECLARE CONTINUE HANDLER FOR SQLWARNING
    SET v_message = 'A warning occurred';

  -- Executable statements
  SELECT COUNT(*) INTO v_count FROM orders WHERE status = 'PENDING';
  SET v_message = CONCAT('Pending orders: ', v_count);

  SELECT v_message AS result;
END$$
DELIMITER ;
```

## Declaration Order Rules

Inside `BEGIN...END`, declarations must appear in this order before any executable statements:

```text
1. DECLARE variable  - local variables
2. DECLARE condition - named conditions
3. DECLARE cursor    - cursors
4. DECLARE handler   - condition handlers
5. Executable SQL statements
```

Placing declarations after executable statements causes a syntax error.

## Nested BEGIN...END Blocks

You can nest `BEGIN...END` blocks to create inner scopes with their own local variables:

```sql
DELIMITER $$
CREATE PROCEDURE nested_blocks()
BEGIN
  DECLARE v_outer INT DEFAULT 10;

  -- Inner block with its own scope
  inner_scope: BEGIN
    DECLARE v_inner INT DEFAULT 20;
    SET v_inner = v_outer + v_inner;
    SELECT v_inner AS inner_result;
  END inner_scope;

  -- v_inner is not accessible here
  SELECT v_outer AS outer_result;
END$$
DELIMITER ;
```

Variables declared in an inner block are not visible in the outer block.

## Using BEGIN...END in Condition Handlers

`BEGIN...END` allows multiple statements inside a handler:

```sql
DELIMITER $$
CREATE PROCEDURE safe_update(IN p_id BIGINT, IN p_status VARCHAR(20))
BEGIN
  DECLARE v_errno INT    DEFAULT 0;
  DECLARE v_msg   TEXT   DEFAULT '';

  DECLARE EXIT HANDLER FOR SQLEXCEPTION
  BEGIN
    -- Multiple statements in the handler
    GET DIAGNOSTICS CONDITION 1
      v_errno = MYSQL_ERRNO,
      v_msg   = MESSAGE_TEXT;

    INSERT INTO error_log (error_code, message, logged_at)
    VALUES (v_errno, v_msg, NOW());

    ROLLBACK;

    SELECT CONCAT('Error ', v_errno, ': ', v_msg) AS error_detail;
  END;

  START TRANSACTION;
    UPDATE orders SET status = p_status WHERE id = p_id;
    IF ROW_COUNT() = 0 THEN
      SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'No order found with given id';
    END IF;
  COMMIT;

  SELECT 'Update successful' AS result;
END$$
DELIMITER ;
```

## BEGIN...END in Triggers

Triggers use `BEGIN...END` whenever they need more than one statement:

```sql
DELIMITER $$
CREATE TRIGGER trg_order_audit
AFTER UPDATE ON orders
FOR EACH ROW
BEGIN
  IF OLD.status <> NEW.status THEN
    INSERT INTO order_audit (order_id, old_status, new_status, changed_at)
    VALUES (NEW.id, OLD.status, NEW.status, NOW());
  END IF;
END$$
DELIMITER ;
```

## BEGIN...END in Events

```sql
DELIMITER $$
CREATE EVENT ev_daily_cleanup
  ON SCHEDULE EVERY 1 DAY
  STARTS CURDATE() + INTERVAL 1 DAY + INTERVAL 2 HOUR
  DO
BEGIN
  DECLARE v_cutoff DATE DEFAULT CURDATE() - INTERVAL 90 DAY;

  DELETE FROM session_log  WHERE created_at < v_cutoff;
  DELETE FROM temp_uploads WHERE created_at < v_cutoff;

  INSERT INTO maintenance_log (task, run_at)
  VALUES ('daily_cleanup', NOW());
END$$
DELIMITER ;
```

## Using DELIMITER Correctly

When creating stored programs, change the delimiter so MySQL does not interpret semicolons inside the body as statement terminators:

```bash
# In mysql CLI
mysql> DELIMITER $$
mysql> CREATE PROCEDURE example() BEGIN SELECT 1; END$$
mysql> DELIMITER ;
```

In application code, use client library methods that support multi-statement stored procedure creation rather than the `DELIMITER` command.

## Summary

`BEGIN...END` compound statements are the core structural element of MySQL stored programs. They group multiple statements, create a local variable scope, and allow condition handler definitions. Declarations must precede executable statements within a block, and blocks can be nested to create inner scopes. Use `BEGIN...END` in procedures, functions, triggers, and events to write organized, maintainable database logic.
