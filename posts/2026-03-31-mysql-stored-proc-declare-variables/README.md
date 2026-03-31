# How to Declare Variables in MySQL Stored Procedures

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Stored Procedure, Variable, Programming, SQL

Description: Learn how to declare and use local variables in MySQL stored procedures, including proper scoping rules, data types, and initialization patterns.

---

Local variables in MySQL stored procedures allow you to store intermediate values, counters, and results during procedure execution. They are declared with the `DECLARE` statement and are only accessible within the `BEGIN...END` block where they are declared.

## Declaring Variables

Variables must be declared at the top of a `BEGIN...END` block, before any other statements:

```sql
DELIMITER //

CREATE PROCEDURE calculate_order_stats(IN p_customer_id INT)
BEGIN
    -- Declare variables at the top
    DECLARE v_order_count INT;
    DECLARE v_total_amount DECIMAL(10,2);
    DECLARE v_avg_order DECIMAL(10,2);
    DECLARE v_customer_name VARCHAR(100);

    -- Assign values after declarations
    SELECT COUNT(*), SUM(total), AVG(total)
    INTO v_order_count, v_total_amount, v_avg_order
    FROM orders
    WHERE customer_id = p_customer_id;

    SELECT name INTO v_customer_name
    FROM customers
    WHERE id = p_customer_id;

    SELECT v_customer_name AS customer,
           v_order_count AS total_orders,
           v_total_amount AS lifetime_value,
           v_avg_order AS avg_order_value;
END //

DELIMITER ;
```

## Variable Declaration Syntax

The full `DECLARE` syntax is:

```sql
DECLARE var_name [, var_name] ... data_type [DEFAULT default_value];
```

Examples of common variable types:

```sql
DELIMITER //

CREATE PROCEDURE demo_variable_types()
BEGIN
    DECLARE v_count INT DEFAULT 0;
    DECLARE v_price DECIMAL(8,2) DEFAULT 0.00;
    DECLARE v_name VARCHAR(255) DEFAULT '';
    DECLARE v_flag BOOLEAN DEFAULT FALSE;
    DECLARE v_created DATE DEFAULT CURDATE();
    DECLARE v_timestamp DATETIME DEFAULT NOW();

    -- Variables without DEFAULT are initialized to NULL
    DECLARE v_nullable INT;

    SET v_count = 10;
    SET v_price = 29.99;
    SET v_name = 'Product A';

    SELECT v_count, v_price, v_name, v_flag, v_nullable;
END //

DELIMITER ;
```

## Variable Scoping Rules

Variables are scoped to the `BEGIN...END` block where they are declared. Nested blocks can access outer variables but not vice versa:

```sql
DELIMITER //

CREATE PROCEDURE scope_example()
BEGIN
    DECLARE v_outer INT DEFAULT 1;

    BEGIN
        -- Inner block can access v_outer
        DECLARE v_inner INT DEFAULT 2;
        SET v_outer = v_outer + v_inner;
        SELECT v_outer, v_inner; -- Both accessible here
    END;

    -- v_inner is NOT accessible here
    SELECT v_outer; -- Shows updated value of 3
END //

DELIMITER ;
```

## Assigning Values to Variables

Use `SET` or `SELECT ... INTO` to assign values:

```sql
DELIMITER //

CREATE PROCEDURE variable_assignment_demo()
BEGIN
    DECLARE v_count INT;
    DECLARE v_max_id INT;

    -- Assign with SET
    SET v_count = 100;

    -- Assign from a query with SELECT INTO
    SELECT MAX(id) INTO v_max_id FROM orders;

    -- Assign multiple variables at once
    SELECT COUNT(*), MAX(id)
    INTO v_count, v_max_id
    FROM orders
    WHERE status = 'pending';

    SELECT v_count AS pending_count, v_max_id AS latest_id;
END //

DELIMITER ;
```

## User-Defined Variables vs. Local Variables

Do not confuse local procedure variables with user-defined session variables:

```sql
-- User-defined variable (session scope, @ prefix)
SET @session_var = 100;

-- Local procedure variable (procedure scope, no @ prefix)
DECLARE v_local_var INT DEFAULT 0;
```

User-defined variables persist for the duration of the session, while local variables exist only within the procedure execution.

## Using Variables in Loops and Conditionals

```sql
DELIMITER //

CREATE PROCEDURE sum_up_to(IN p_limit INT, OUT p_result INT)
BEGIN
    DECLARE v_i INT DEFAULT 1;
    DECLARE v_sum INT DEFAULT 0;

    WHILE v_i <= p_limit DO
        SET v_sum = v_sum + v_i;
        SET v_i = v_i + 1;
    END WHILE;

    SET p_result = v_sum;
END //

DELIMITER ;

CALL sum_up_to(10, @result);
SELECT @result; -- Returns 55
```

## Summary

Declare local variables with `DECLARE` at the top of a `BEGIN...END` block before any other statements. Specify a data type and optional `DEFAULT` value. Variables without a default are `NULL`. Assign values using `SET` or `SELECT ... INTO`. Local variables are scoped to their `BEGIN...END` block and do not persist after the procedure completes, unlike session-level user-defined variables with the `@` prefix.
