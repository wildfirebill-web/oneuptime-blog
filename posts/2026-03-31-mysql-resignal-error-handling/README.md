# How to Use RESIGNAL in MySQL Error Handling

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, RESIGNAL, Error Handling, Stored Procedure

Description: Learn how to use RESIGNAL in MySQL stored procedures to re-raise caught exceptions with modified or original error details, enabling layered error handling.

---

## What Is RESIGNAL

`RESIGNAL` is used inside a condition handler in a MySQL stored procedure or function to re-raise the current exception. It allows you to catch an error, optionally perform cleanup or logging, and then propagate the error up to the calling context rather than silently swallowing it.

Without `RESIGNAL`, a `CONTINUE HANDLER` catches an error and execution continues. With `RESIGNAL`, you re-throw the error so that it propagates to the caller.

## Basic RESIGNAL

Re-raise the caught exception unchanged:

```sql
DELIMITER $$
CREATE PROCEDURE delete_user(IN p_user_id BIGINT)
BEGIN
  DECLARE EXIT HANDLER FOR SQLEXCEPTION
  BEGIN
    -- Log before propagating
    INSERT INTO error_log (message, logged_at)
    VALUES (CONCAT('delete_user failed for user_id=', p_user_id), NOW());

    RESIGNAL;  -- re-raise original error to caller
  END;

  DELETE FROM user_sessions WHERE user_id = p_user_id;
  DELETE FROM users WHERE id = p_user_id;
END$$
DELIMITER ;
```

The caller will see the original MySQL error just as if the stored procedure had no handler.

## RESIGNAL with Modified Error Information

You can change the error code or message before re-raising:

```sql
DELIMITER $$
CREATE PROCEDURE create_order(
  IN p_customer_id BIGINT,
  IN p_total       DECIMAL(10,2)
)
BEGIN
  DECLARE v_msg TEXT;

  DECLARE EXIT HANDLER FOR SQLEXCEPTION
  BEGIN
    GET DIAGNOSTICS CONDITION 1 v_msg = MESSAGE_TEXT;

    -- Re-raise with a custom application error code and message
    RESIGNAL SQLSTATE '45000'
      SET MESSAGE_TEXT = CONCAT('create_order failed: ', v_msg),
          MYSQL_ERRNO  = 5001;
  END;

  INSERT INTO orders (customer_id, total, status)
  VALUES (p_customer_id, p_total, 'PENDING');
END$$
DELIMITER ;
```

The caller now receives SQLSTATE `45000` (generic user-defined error) with a descriptive message.

## Difference Between RESIGNAL and SIGNAL

```text
SIGNAL   - raise a new error from scratch (can be used anywhere)
RESIGNAL - re-raise the currently handled exception (only inside a handler)
           RESIGNAL with no arguments re-raises the original error unchanged
           RESIGNAL with arguments re-raises with modified attributes
```

## Layered Error Handling Pattern

In multi-level stored procedure calls, each layer can add context:

```sql
DELIMITER $$

CREATE PROCEDURE inner_proc()
BEGIN
  DECLARE EXIT HANDLER FOR SQLEXCEPTION
  BEGIN
    RESIGNAL SQLSTATE '45000'
      SET MESSAGE_TEXT = 'inner_proc: foreign key violation detected';
  END;

  INSERT INTO orders (customer_id, total) VALUES (99999, 100.00);
END$$

CREATE PROCEDURE outer_proc()
BEGIN
  DECLARE v_msg TEXT;

  DECLARE EXIT HANDLER FOR SQLEXCEPTION
  BEGIN
    GET DIAGNOSTICS CONDITION 1 v_msg = MESSAGE_TEXT;

    INSERT INTO error_log (layer, message, logged_at)
    VALUES ('outer_proc', v_msg, NOW());

    RESIGNAL;  -- propagate to the top-level caller
  END;

  CALL inner_proc();
  -- other operations
END$$

DELIMITER ;
```

## RESIGNAL with GET STACKED DIAGNOSTICS

Inside a handler, the diagnostics area may be modified by logging statements. Use `GET STACKED DIAGNOSTICS` to read the original error before it is overwritten:

```sql
DECLARE EXIT HANDLER FOR SQLEXCEPTION
BEGIN
  DECLARE v_original_msg TEXT;

  -- Read the original error BEFORE any other statements clear it
  GET STACKED DIAGNOSTICS CONDITION 1
    v_original_msg = MESSAGE_TEXT;

  -- Now safe to run other statements that might change diagnostics
  INSERT INTO error_log (message) VALUES (v_original_msg);

  -- Re-raise with original message intact
  RESIGNAL SQLSTATE '45000'
    SET MESSAGE_TEXT = v_original_msg;
END;
```

## Complete Example with Rollback

```sql
DELIMITER $$
CREATE PROCEDURE transfer_funds(
  IN p_from_account BIGINT,
  IN p_to_account   BIGINT,
  IN p_amount       DECIMAL(10,2)
)
BEGIN
  DECLARE EXIT HANDLER FOR SQLEXCEPTION
  BEGIN
    ROLLBACK;
    RESIGNAL;
  END;

  START TRANSACTION;
    UPDATE accounts SET balance = balance - p_amount WHERE id = p_from_account;
    UPDATE accounts SET balance = balance + p_amount WHERE id = p_to_account;
  COMMIT;
END$$
DELIMITER ;
```

If either UPDATE fails, the transaction is rolled back and the original error is propagated to the caller.

## Summary

`RESIGNAL` enables layered error handling in MySQL stored procedures. Use it without arguments to re-raise the original exception unchanged, or supply a SQLSTATE and attributes to wrap the error with additional context. Combine `RESIGNAL` with `GET STACKED DIAGNOSTICS` to safely read the original error before any logging statements modify the diagnostics area. Always roll back open transactions in the handler before calling `RESIGNAL` to avoid leaving partial data in the database.
