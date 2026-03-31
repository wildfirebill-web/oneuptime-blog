# How to Use OUT Parameters in MySQL Stored Procedures

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Mysql, Stored Procedures, Parameters, Sql

Description: Learn how to use OUT and INOUT parameters in MySQL stored procedures to return values to the caller without using a SELECT result set.

---

## Parameter Types in MySQL Stored Procedures

MySQL stored procedures support three parameter modes:

| Mode | Description |
|---|---|
| `IN` | Input only - value passed into procedure, changes not visible to caller |
| `OUT` | Output only - initial value is NULL inside procedure, caller receives final value |
| `INOUT` | Both - caller passes a value in and receives the modified value out |

## Creating a Procedure with an OUT Parameter

```sql
DELIMITER //

CREATE PROCEDURE GetCustomerOrderCount(
  IN cust_id INT,
  OUT order_count INT
)
BEGIN
  SELECT COUNT(*)
  INTO order_count
  FROM orders
  WHERE customer_id = cust_id;
END //

DELIMITER ;
```

## Calling a Procedure with OUT Parameters

OUT parameters require user-defined variables prefixed with `@`:

```sql
CALL GetCustomerOrderCount(42, @count);
SELECT @count AS order_count;
```

```text
+-------------+
| order_count |
+-------------+
| 15          |
+-------------+
```

## OUT Parameter with Multiple Return Values

```sql
DELIMITER //

CREATE PROCEDURE GetCustomerStats(
  IN cust_id INT,
  OUT order_count INT,
  OUT total_amount DECIMAL(12, 2),
  OUT last_order_date DATE
)
BEGIN
  SELECT
    COUNT(*),
    COALESCE(SUM(total_amount), 0),
    MAX(order_date)
  INTO order_count, total_amount, last_order_date
  FROM orders
  WHERE customer_id = cust_id;
END //

DELIMITER ;
```

```sql
CALL GetCustomerStats(42, @cnt, @total, @last_date);
SELECT @cnt, @total, @last_date;
```

## Using INOUT Parameters

`INOUT` allows you to pass a value in and receive a modified value back:

```sql
DELIMITER //

CREATE PROCEDURE ApplyDiscount(
  INOUT price DECIMAL(10, 2),
  IN discount_pct DECIMAL(5, 2)
)
BEGIN
  SET price = price - (price * discount_pct / 100);
END //

DELIMITER ;
```

```sql
SET @price = 100.00;
CALL ApplyDiscount(@price, 15);
SELECT @price AS discounted_price;
```

```text
+------------------+
| discounted_price |
+------------------+
| 85.00            |
+------------------+
```

## OUT Parameters vs. SELECT Result Sets

When to use OUT parameters vs. returning a result set:

| Use Case | Preferred Approach |
|---|---|
| Single scalar value needed by application | OUT parameter |
| Status code or error flag | OUT parameter |
| Multiple rows of data | SELECT result set |
| Chaining procedures | INOUT or OUT parameter |

## Checking OUT Parameter Value Before Assignment

An OUT parameter starts as NULL inside the procedure:

```sql
DELIMITER //

CREATE PROCEDURE CheckAndCount(
  IN table_name_in VARCHAR(64),
  OUT row_count INT
)
BEGIN
  -- row_count is NULL here initially
  SET row_count = 0;

  SELECT COUNT(*)
  INTO row_count
  FROM information_schema.TABLES
  WHERE TABLE_SCHEMA = DATABASE()
    AND TABLE_NAME = table_name_in;
END //

DELIMITER ;
```

## Using OUT Parameters in Application Code

Example in Python using MySQL Connector:

```python
import mysql.connector

conn = mysql.connector.connect(
    host='localhost', user='app', password='secret', database='shop'
)
cursor = conn.cursor()

cursor.callproc('GetCustomerStats', [42, 0, 0.0, None])

# Fetch OUT parameter values
for result in cursor.stored_results():
    pass

# Get output parameters
cursor.execute("SELECT @_GetCustomerStats_1, @_GetCustomerStats_2, @_GetCustomerStats_3")
row = cursor.fetchone()
print(f"Count: {row[0]}, Total: {row[1]}, Last Order: {row[2]}")
```

## Summary

OUT parameters in MySQL stored procedures allow procedures to return scalar values to the caller using user-defined session variables (`@variable`). Use OUT for returning a single computed value or status, INOUT for modifying a value in place, and SELECT result sets for returning rows. OUT parameters are especially useful when chaining procedure calls or when the application needs a status code alongside data.
