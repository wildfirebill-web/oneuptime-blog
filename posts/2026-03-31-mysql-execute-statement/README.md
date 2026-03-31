# How to Use EXECUTE Statement in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Prepared Statement, Query, Dynamic SQL, Security

Description: Learn how to use the MySQL EXECUTE statement to run prepared statements with bound parameters, including passing values via user variables.

---

## What is the EXECUTE Statement?

`EXECUTE` runs a previously prepared statement by name. It is the second step in MySQL's server-side prepared statement workflow: first you `PREPARE` the statement, then you `EXECUTE` it (optionally multiple times with different parameters), and finally you `DEALLOCATE PREPARE` to free resources.

## Basic Syntax

```sql
EXECUTE stmt_name [USING @var1 [, @var2] ...];
```

The `USING` clause passes user variables as the values for `?` placeholders in the prepared statement.

## Simple Example

```sql
-- Step 1: Prepare
PREPARE get_user FROM 'SELECT id, name FROM users WHERE id = ?';

-- Step 2: Assign the parameter value to a user variable
SET @user_id = 42;

-- Step 3: Execute
EXECUTE get_user USING @user_id;

-- Step 4: Execute again with a different value
SET @user_id = 99;
EXECUTE get_user USING @user_id;
```

## Using Multiple Parameters

```sql
PREPARE search_orders FROM
  'SELECT * FROM orders WHERE customer_id = ? AND status = ?';

SET @cust_id = 10;
SET @status = 'pending';

EXECUTE search_orders USING @cust_id, @status;
```

The order of `USING` variables must match the order of `?` placeholders.

## Executing DML Statements

Prepared statements are not limited to `SELECT`. You can use them with `INSERT`, `UPDATE`, and `DELETE`:

```sql
PREPARE update_balance FROM
  'UPDATE accounts SET balance = balance + ? WHERE account_id = ?';

SET @amount = 500.00;
SET @account_id = 1001;

EXECUTE update_balance USING @amount, @account_id;
```

## Executing in a Loop (Stored Procedure)

The real power of `EXECUTE` is running the same prepared statement many times efficiently inside stored procedures:

```sql
DELIMITER //
CREATE PROCEDURE bulk_update_prices(IN multiplier DECIMAL(5,2))
BEGIN
  DECLARE done INT DEFAULT 0;
  DECLARE pid INT;
  DECLARE cur CURSOR FOR SELECT id FROM products;
  DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = 1;

  PREPARE stmt FROM 'UPDATE products SET price = price * ? WHERE id = ?';

  OPEN cur;
  read_loop: LOOP
    FETCH cur INTO pid;
    IF done THEN LEAVE read_loop; END IF;
    SET @mult = multiplier;
    SET @id = pid;
    EXECUTE stmt USING @mult, @id;
  END LOOP;

  CLOSE cur;
  DEALLOCATE PREPARE stmt;
END//
DELIMITER ;
```

## Checking the Row Count After Execute

After executing a DML statement, check `ROW_COUNT()` to see how many rows were affected:

```sql
PREPARE del_old FROM 'DELETE FROM logs WHERE created_at < ?';
SET @cutoff = DATE_SUB(NOW(), INTERVAL 90 DAY);
EXECUTE del_old USING @cutoff;
SELECT ROW_COUNT() AS deleted_rows;
DEALLOCATE PREPARE del_old;
```

## Limitations of EXECUTE

- Parameters (`?`) can only represent data values, not table names, column names, or keywords.
- Prepared statements exist only within the current session.
- The `USING` clause only accepts user variables (prefixed with `@`), not literals.

```sql
-- ERROR: cannot use a literal directly in USING
EXECUTE get_user USING 42;

-- CORRECT: assign to a user variable first
SET @id = 42;
EXECUTE get_user USING @id;
```

## Summary

`EXECUTE` runs a prepared statement by name, substituting `?` placeholders with the values of user variables passed via `USING`. It enables efficient repeated execution of the same query pattern with different parameters, and is the foundation of safe parameterized queries in MySQL's procedural SQL. Always pair each `PREPARE` with a corresponding `DEALLOCATE PREPARE` to release server resources.
