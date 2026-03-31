# How to Use ITERATE and LEAVE in MySQL Loops

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Stored Procedure, Loop, ITERATE, LEAVE

Description: Learn how to control MySQL loop flow using ITERATE (continue) and LEAVE (break) inside LOOP, WHILE, and REPEAT constructs with practical examples.

---

## Loop Constructs in MySQL

MySQL stored procedures support three loop types:
- `LOOP` - infinite loop, must use `LEAVE` to exit
- `WHILE` - tests condition before each iteration
- `REPEAT` - tests condition after each iteration (do-while style)

`ITERATE` restarts the loop from the top (equivalent to `continue` in other languages). `LEAVE` exits the loop entirely (equivalent to `break`).

Both require a loop label to identify which loop to affect.

## Basic LOOP with LEAVE

```sql
DELIMITER $$

CREATE PROCEDURE count_to_ten()
BEGIN
  DECLARE counter INT DEFAULT 0;

  count_loop: LOOP
    SET counter = counter + 1;

    IF counter = 10 THEN
      LEAVE count_loop;   -- exit when counter reaches 10
    END IF;

    SELECT counter;       -- print current value
  END LOOP count_loop;
END$$

DELIMITER ;
```

Without `LEAVE`, the loop would run forever. The label `count_loop:` is required for `LEAVE` to know which loop to exit.

## ITERATE to Skip Iterations

`ITERATE` jumps back to the start of the loop, skipping any remaining statements in the current iteration:

```sql
DELIMITER $$

CREATE PROCEDURE print_odd_numbers(IN p_max INT)
BEGIN
  DECLARE i INT DEFAULT 0;

  num_loop: LOOP
    SET i = i + 1;

    IF i > p_max THEN
      LEAVE num_loop;       -- exit when done
    END IF;

    IF MOD(i, 2) = 0 THEN
      ITERATE num_loop;     -- skip even numbers, go to next iteration
    END IF;

    SELECT i AS odd_number; -- only reaches here for odd numbers
  END LOOP num_loop;
END$$

DELIMITER ;
```

## WHILE Loop with LEAVE and ITERATE

```sql
DELIMITER $$

CREATE PROCEDURE process_batch()
BEGIN
  DECLARE batch_id   INT DEFAULT 1;
  DECLARE max_batch  INT DEFAULT 100;
  DECLARE row_count  INT;

  batch_loop: WHILE batch_id <= max_batch DO

    -- Check if this batch has data
    SELECT COUNT(*) INTO row_count
    FROM staging_table
    WHERE batch = batch_id AND processed = 0;

    IF row_count = 0 THEN
      SET batch_id = batch_id + 1;
      ITERATE batch_loop;   -- skip empty batches
    END IF;

    -- Process the batch
    UPDATE staging_table
    SET processed = 1
    WHERE batch = batch_id AND processed = 0;

    -- Stop early if an unexpected condition occurs
    IF ROW_COUNT() = 0 THEN
      LEAVE batch_loop;     -- something went wrong, exit
    END IF;

    SET batch_id = batch_id + 1;
  END WHILE batch_loop;
END$$

DELIMITER ;
```

## REPEAT Loop Example

`REPEAT` checks the condition at the end, so the body always runs at least once:

```sql
DELIMITER $$

CREATE PROCEDURE retry_until_success()
BEGIN
  DECLARE attempts  INT DEFAULT 0;
  DECLARE succeeded INT DEFAULT 0;

  retry_loop: REPEAT
    SET attempts = attempts + 1;

    -- Simulate operation that may fail
    BEGIN
      -- Try to acquire a lock or call external logic
      IF attempts >= 3 THEN
        SET succeeded = 1;
      END IF;
    END;

    IF succeeded = 1 THEN
      LEAVE retry_loop;   -- exit as soon as it succeeds
    END IF;

  UNTIL attempts >= 5     -- also stop after 5 attempts regardless
  END REPEAT retry_loop;

  SELECT attempts AS total_attempts, succeeded;
END$$

DELIMITER ;
```

## Nested Loops with Labels

Labels are essential when nesting loops, because `LEAVE` and `ITERATE` must specify which loop they target:

```sql
DELIMITER $$

CREATE PROCEDURE nested_loop_example()
BEGIN
  DECLARE i INT DEFAULT 0;
  DECLARE j INT DEFAULT 0;

  outer_loop: LOOP
    SET i = i + 1;
    SET j = 0;

    IF i > 3 THEN
      LEAVE outer_loop;       -- exit the outer loop
    END IF;

    inner_loop: LOOP
      SET j = j + 1;

      IF j > 3 THEN
        LEAVE inner_loop;     -- exit only the inner loop
      END IF;

      IF j = 2 THEN
        ITERATE inner_loop;   -- skip j=2 in the inner loop
      END IF;

      SELECT CONCAT('i=', i, ', j=', j) AS pair;
    END LOOP inner_loop;

  END LOOP outer_loop;
END$$

DELIMITER ;
```

## Summary

MySQL loop control uses `ITERATE` (continue to next iteration) and `LEAVE` (exit the loop) paired with mandatory loop labels. `LOOP` requires `LEAVE` to terminate, while `WHILE` and `REPEAT` can exit naturally via their conditions or early via `LEAVE`. Use labels on nested loops to make it explicit which loop an `ITERATE` or `LEAVE` targets. These constructs are available only inside stored procedures and functions, not in plain SQL statements.
