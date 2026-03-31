# How to Use IN Parameters in MySQL Stored Procedures

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Stored Procedure, Parameter, SQL, Database

Description: Learn how to define and use IN parameters in MySQL stored procedures to pass input values and write reusable, parameterized queries.

---

## What Are IN Parameters?

In MySQL stored procedures, `IN` is the default parameter mode. An `IN` parameter passes a value into the procedure. The procedure can read it, but any changes made to the parameter inside the procedure body are not visible to the caller. This makes `IN` parameters ideal for filtering conditions, IDs, flags, and other read-only inputs.

## Declaring an IN Parameter

```sql
DELIMITER $$
CREATE PROCEDURE procedure_name(IN param_name datatype)
BEGIN
    -- use param_name here
END$$
DELIMITER ;
```

Because `IN` is the default, the following two declarations are equivalent:

```sql
CREATE PROCEDURE foo(IN p_id INT)   -- explicit
CREATE PROCEDURE foo(p_id INT)      -- implicit
```

## Example: Filter Rows by a Single IN Parameter

```sql
DELIMITER $$
CREATE PROCEDURE get_orders_by_status(IN p_status VARCHAR(20))
BEGIN
    SELECT id, customer_id, total, created_at
    FROM orders
    WHERE status = p_status;
END$$
DELIMITER ;

CALL get_orders_by_status('pending');
CALL get_orders_by_status('shipped');
```

## Example: Multiple IN Parameters

```sql
DELIMITER $$
CREATE PROCEDURE get_orders_in_range(
    IN p_customer_id INT,
    IN p_start_date  DATE,
    IN p_end_date    DATE
)
BEGIN
    SELECT id, total, status, created_at
    FROM orders
    WHERE customer_id = p_customer_id
      AND created_at  BETWEEN p_start_date AND p_end_date
    ORDER BY created_at DESC;
END$$
DELIMITER ;

CALL get_orders_in_range(42, '2025-01-01', '2025-03-31');
```

## IN Parameters Are Read-Only to the Caller

Modifications inside the procedure do not affect the original variable:

```sql
DELIMITER $$
CREATE PROCEDURE demo_readonly(IN p_value INT)
BEGIN
    SET p_value = p_value * 10;  -- change is local only
    SELECT p_value;              -- shows modified value inside procedure
END$$
DELIMITER ;

SET @my_val = 5;
CALL demo_readonly(@my_val);
SELECT @my_val;  -- still 5
```

## Using IN Parameters with INSERT and UPDATE

```sql
DELIMITER $$
CREATE PROCEDURE create_user(
    IN p_name  VARCHAR(100),
    IN p_email VARCHAR(255),
    IN p_role  VARCHAR(50)
)
BEGIN
    INSERT INTO users (name, email, role, created_at)
    VALUES (p_name, p_email, p_role, NOW());
END$$
DELIMITER ;

CALL create_user('Alice', 'alice@example.com', 'admin');
```

## Passing NULL as an IN Parameter

IN parameters accept NULL values, which you can test with `IS NULL`:

```sql
DELIMITER $$
CREATE PROCEDURE search_products(IN p_category VARCHAR(50))
BEGIN
    IF p_category IS NULL THEN
        SELECT * FROM products;
    ELSE
        SELECT * FROM products WHERE category = p_category;
    END IF;
END$$
DELIMITER ;

CALL search_products(NULL);        -- returns all products
CALL search_products('electronics'); -- filtered
```

## Calling from Application Code

### Python

```python
import mysql.connector

conn = mysql.connector.connect(host="localhost", user="app", password="secret", database="shop")
cursor = conn.cursor()
cursor.callproc("get_orders_by_status", ["pending"])
for result in cursor.stored_results():
    print(result.fetchall())
cursor.close()
conn.close()
```

## Summary

`IN` parameters let you pass read-only values into a MySQL stored procedure. They are the most common parameter type and behave like function arguments in general-purpose programming languages. Use them to parameterize queries, insert statements, and conditional logic inside your procedures.
