# MySQL Stored Procedure Syntax Cheat Sheet

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Stored Procedure, Syntax, Cheat Sheet

Description: Quick reference for MySQL stored procedure syntax including CREATE, variables, IF/ELSE, loops, cursors, error handling, and calling conventions with code examples.

---

## Create and Drop

```sql
DELIMITER $$

CREATE PROCEDURE procedure_name(
  IN  p_input  INT,
  OUT p_output VARCHAR(100),
  INOUT p_both INT
)
BEGIN
  -- procedure body
END $$

DELIMITER ;

DROP PROCEDURE IF EXISTS procedure_name;
```

## Calling a Procedure

```sql
CALL procedure_name(42, @result, @value);
SELECT @result;
```

## Variables

```sql
DECLARE v_count INT DEFAULT 0;
DECLARE v_name  VARCHAR(100);

SET v_name = 'Alice';
SELECT COUNT(*) INTO v_count FROM orders WHERE customer_id = 1;
```

## IF / ELSEIF / ELSE

```sql
IF v_count > 100 THEN
  SET p_output = 'Heavy user';
ELSEIF v_count > 10 THEN
  SET p_output = 'Regular user';
ELSE
  SET p_output = 'Light user';
END IF;
```

## CASE Statement

```sql
CASE v_status
  WHEN 'A' THEN SET v_label = 'Active';
  WHEN 'I' THEN SET v_label = 'Inactive';
  ELSE          SET v_label = 'Unknown';
END CASE;
```

## WHILE Loop

```sql
SET v_i = 1;
WHILE v_i <= 10 DO
  INSERT INTO log_table VALUES (v_i);
  SET v_i = v_i + 1;
END WHILE;
```

## REPEAT Loop

```sql
REPEAT
  SET v_i = v_i + 1;
UNTIL v_i > 10 END REPEAT;
```

## LOOP with LEAVE

```sql
my_loop: LOOP
  IF v_i > 10 THEN
    LEAVE my_loop;
  END IF;
  SET v_i = v_i + 1;
END LOOP my_loop;
```

## Cursors

```sql
DECLARE done INT DEFAULT FALSE;
DECLARE v_id INT;

DECLARE cur CURSOR FOR
  SELECT id FROM customers WHERE active = 1;

DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = TRUE;

OPEN cur;
read_loop: LOOP
  FETCH cur INTO v_id;
  IF done THEN
    LEAVE read_loop;
  END IF;
  -- process v_id
END LOOP;
CLOSE cur;
```

## Error Handling

```sql
DECLARE EXIT HANDLER FOR SQLEXCEPTION
BEGIN
  ROLLBACK;
  RESIGNAL;
END;

DECLARE CONTINUE HANDLER FOR SQLSTATE '23000'
BEGIN
  SET v_error = 1;
END;
```

## Transactions Inside Procedures

```sql
START TRANSACTION;
  INSERT INTO orders ...;
  UPDATE inventory ...;
  IF v_error THEN
    ROLLBACK;
  ELSE
    COMMIT;
  END IF;
```

## Returning Data

```sql
-- Via OUT parameter
SET p_output = v_result;

-- Via SELECT (returns result set to caller)
SELECT id, name FROM customers WHERE active = 1;
```

## Show All Procedures

```sql
SHOW PROCEDURE STATUS WHERE db = 'mydb';
SHOW CREATE PROCEDURE procedure_name;
```

## Summary

MySQL stored procedures use DELIMITER to separate the procedure body from the client terminator. Variables are declared with DECLARE, and flow control uses IF/CASE/WHILE/LOOP. Cursors iterate over result sets row by row. DECLARE HANDLER catches errors for graceful recovery. Procedures return data via OUT parameters or by executing a SELECT - the latter sends result sets back to the calling application.
