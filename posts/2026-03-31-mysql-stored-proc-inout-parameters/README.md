# How to Use INOUT Parameters in MySQL Stored Procedures

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Stored Procedure, Parameter, SQL, Database

Description: Learn how to use INOUT parameters in MySQL stored procedures to pass a value in, modify it, and return the updated value to the caller.

---

## What Are INOUT Parameters?

An `INOUT` parameter in a MySQL stored procedure acts as both an input and an output. The caller provides an initial value, the procedure reads and modifies it, and the modified value is returned to the caller. This is useful when you need to transform a value in place without requiring a separate return variable.

## Syntax

```sql
DELIMITER $$
CREATE PROCEDURE procedure_name(INOUT param_name datatype)
BEGIN
    -- read and modify param_name
END$$
DELIMITER ;
```

Always pass a session variable (prefixed with `@`) when calling a procedure that has `INOUT` parameters, because MySQL writes the result back through that variable.

## Basic Example: Doubling a Number

```sql
DELIMITER $$
CREATE PROCEDURE double_value(INOUT p_num INT)
BEGIN
    SET p_num = p_num * 2;
END$$
DELIMITER ;

SET @x = 7;
CALL double_value(@x);
SELECT @x;  -- Returns 14
```

## Practical Example: Apply a Percentage Discount

```sql
DELIMITER $$
CREATE PROCEDURE apply_discount(
    INOUT p_price   DECIMAL(10,2),
    IN    p_percent INT
)
BEGIN
    SET p_price = ROUND(p_price - (p_price * p_percent / 100), 2);
END$$
DELIMITER ;

SET @price = 120.00;
CALL apply_discount(@price, 10);
SELECT @price;  -- Returns 108.00
```

## Chaining Multiple Transformations

Because INOUT parameters reflect the final state after the procedure returns, you can call procedures in sequence to chain transformations:

```sql
SET @amount = 500.00;

-- Apply 10% discount
CALL apply_discount(@amount, 10);
SELECT @amount;  -- 450.00

-- Apply another 5% discount
CALL apply_discount(@amount, 5);
SELECT @amount;  -- 427.50
```

## Example: Accumulate a Running Total

```sql
DELIMITER $$
CREATE PROCEDURE add_to_total(INOUT p_total DECIMAL(12,2), IN p_amount DECIMAL(12,2))
BEGIN
    SET p_total = p_total + p_amount;
END$$
DELIMITER ;

SET @running_total = 0.00;
CALL add_to_total(@running_total, 100.00);
CALL add_to_total(@running_total, 250.00);
CALL add_to_total(@running_total, 75.50);
SELECT @running_total;  -- Returns 425.50
```

## INOUT vs IN vs OUT

| Mode | Caller Sends Value | Procedure Reads It | Procedure Modifies It | Caller Sees Change |
|---|---|---|---|---|
| IN | Yes | Yes | Local only | No |
| OUT | No (ignored) | No initial value | Yes | Yes |
| INOUT | Yes | Yes | Yes | Yes |

Use `INOUT` when the transformed value depends on the original input. Use `OUT` when the procedure computes something entirely new.

## Calling from Application Code

### Python (mysql-connector-python)

```python
import mysql.connector

conn = mysql.connector.connect(
    host="localhost", user="app_user", password="secret", database="shop"
)
cursor = conn.cursor()

# Set the session variable first
cursor.execute("SET @price = %s", (120.00,))
cursor.execute("CALL apply_discount(@price, %s)", (10,))
cursor.execute("SELECT @price")
result = cursor.fetchone()
print(result[0])  # 108.0

cursor.close()
conn.close()
```

## Common Pitfalls

- Forgetting to use a `@` session variable means the modified value is lost after the procedure returns.
- Passing a literal directly (e.g., `CALL double_value(7)`) causes a syntax error because literals cannot be written back to.
- Ensure the data type of the session variable matches the parameter type to avoid silent truncation.

## Summary

`INOUT` parameters in MySQL stored procedures allow bidirectional value passing: the caller supplies an initial value, the procedure modifies it, and the updated value is returned. Always use session variables (prefixed with `@`) when calling procedures with `INOUT` parameters. This mode is ideal for in-place transformations like applying discounts, accumulating totals, or encoding/decoding values.
