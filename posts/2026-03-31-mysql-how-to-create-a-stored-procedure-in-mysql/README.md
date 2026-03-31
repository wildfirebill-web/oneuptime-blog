# How to Create a Stored Procedure in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Stored Procedure, SQL, Database Programming

Description: Learn how to create, call, and manage stored procedures in MySQL with practical examples covering parameters, variables, and control flow.

---

## What Is a Stored Procedure?

A stored procedure is a named, reusable block of SQL statements stored in the MySQL database. Stored procedures reduce network round-trips, centralize business logic in the database, and can accept parameters to handle dynamic data.

## Basic Syntax

```sql
DELIMITER //

CREATE PROCEDURE procedure_name (parameters)
BEGIN
  -- SQL statements
END //

DELIMITER ;
```

The `DELIMITER` command changes the statement terminator from `;` to `//` so that MySQL does not interpret the semicolons inside the procedure body as the end of the `CREATE PROCEDURE` statement.

## Creating a Simple Stored Procedure

```sql
DELIMITER //

CREATE PROCEDURE GetAllCustomers()
BEGIN
  SELECT id, name, email FROM customers ORDER BY name;
END //

DELIMITER ;
```

Call the procedure:

```sql
CALL GetAllCustomers();
```

## Stored Procedure with an IN Parameter

```sql
DELIMITER //

CREATE PROCEDURE GetOrdersByCustomer(IN customer_id INT)
BEGIN
  SELECT
    o.id,
    o.order_date,
    o.total_amount
  FROM orders o
  WHERE o.customer_id = customer_id
  ORDER BY o.order_date DESC;
END //

DELIMITER ;
```

```sql
CALL GetOrdersByCustomer(42);
```

## Using Local Variables

Declare variables with `DECLARE` inside the `BEGIN...END` block:

```sql
DELIMITER //

CREATE PROCEDURE SummarizeCustomer(IN cust_id INT)
BEGIN
  DECLARE order_count INT DEFAULT 0;
  DECLARE total_spent DECIMAL(10, 2) DEFAULT 0.00;

  SELECT COUNT(*), SUM(total_amount)
  INTO order_count, total_spent
  FROM orders
  WHERE customer_id = cust_id;

  SELECT order_count AS orders, total_spent AS total;
END //

DELIMITER ;
```

```sql
CALL SummarizeCustomer(42);
```

## Viewing Existing Stored Procedures

```sql
-- List all procedures in a database
SHOW PROCEDURE STATUS WHERE Db = 'your_database';

-- View the definition of a procedure
SHOW CREATE PROCEDURE GetAllCustomers;
```

## Dropping a Stored Procedure

```sql
DROP PROCEDURE IF EXISTS GetAllCustomers;
```

## Altering a Stored Procedure

MySQL does not support `ALTER PROCEDURE` to change the body. Drop and recreate:

```sql
DROP PROCEDURE IF EXISTS GetOrdersByCustomer;

DELIMITER //

CREATE PROCEDURE GetOrdersByCustomer(IN customer_id INT)
BEGIN
  SELECT
    o.id,
    o.order_date,
    o.total_amount,
    o.status
  FROM orders o
  WHERE o.customer_id = customer_id
  ORDER BY o.order_date DESC;
END //

DELIMITER ;
```

## Granting Execute Permission

```sql
GRANT EXECUTE ON PROCEDURE your_database.GetAllCustomers TO 'app_user'@'%';
```

This allows the user to call the procedure without having direct table access.

## Summary

Stored procedures encapsulate SQL logic in the database, making it reusable and callable with `CALL`. Use `DECLARE` for local variables, `IN` parameters for input, and `DELIMITER` to define multi-statement bodies. Manage procedures with `SHOW PROCEDURE STATUS`, `SHOW CREATE PROCEDURE`, and `DROP PROCEDURE`.
