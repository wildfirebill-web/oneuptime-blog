# How to Call a Stored Procedure in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Stored Procedure, SQL, Database, Query

Description: Learn how to call a stored procedure in MySQL using the CALL statement, pass parameters, and retrieve output values.

---

## Overview

Once a stored procedure is created in MySQL, you execute it with the `CALL` statement. You can pass IN, OUT, and INOUT parameters depending on how the procedure is defined. Understanding how to call procedures correctly is essential for working with MySQL's procedural layer.

## Basic CALL Syntax

```sql
CALL procedure_name(arg1, arg2, ...);
```

If the procedure takes no arguments, use empty parentheses:

```sql
CALL procedure_name();
```

## Calling a Procedure with IN Parameters

`IN` parameters are values you supply to the procedure. They are read-only inside the procedure body.

```sql
-- Procedure definition
DELIMITER $$
CREATE PROCEDURE get_orders_by_customer(IN p_customer_id INT)
BEGIN
    SELECT id, total, created_at
    FROM orders
    WHERE customer_id = p_customer_id;
END$$
DELIMITER ;

-- Call the procedure
CALL get_orders_by_customer(42);
```

## Calling a Procedure with OUT Parameters

`OUT` parameters return values from the procedure back to the caller. Pass a user-defined variable prefixed with `@`:

```sql
-- Procedure definition
DELIMITER $$
CREATE PROCEDURE count_orders(IN p_customer_id INT, OUT p_count INT)
BEGIN
    SELECT COUNT(*) INTO p_count
    FROM orders
    WHERE customer_id = p_customer_id;
END$$
DELIMITER ;

-- Call and retrieve the output
CALL count_orders(42, @order_count);
SELECT @order_count;
```

The variable `@order_count` holds the result after the procedure returns.

## Calling a Procedure with INOUT Parameters

`INOUT` parameters are both sent to the procedure and modified by it:

```sql
DELIMITER $$
CREATE PROCEDURE apply_discount(INOUT p_price DECIMAL(10,2), IN p_pct INT)
BEGIN
    SET p_price = p_price - (p_price * p_pct / 100);
END$$
DELIMITER ;

SET @price = 100.00;
CALL apply_discount(@price, 15);
SELECT @price;  -- Returns 85.00
```

## Calling a Procedure from Application Code

### Python (mysql-connector-python)

```python
import mysql.connector

conn = mysql.connector.connect(
    host="localhost", user="app_user", password="secret", database="shop"
)
cursor = conn.cursor()

cursor.callproc("get_orders_by_customer", [42])

for result in cursor.stored_results():
    for row in result.fetchall():
        print(row)

cursor.close()
conn.close()
```

### Node.js (mysql2)

```javascript
const mysql = require("mysql2/promise");

async function main() {
  const conn = await mysql.createConnection({
    host: "localhost",
    user: "app_user",
    password: "secret",
    database: "shop",
  });

  const [rows] = await conn.execute("CALL get_orders_by_customer(?)", [42]);
  console.log(rows[0]); // First result set

  await conn.end();
}

main();
```

## Checking Procedure Existence Before Calling

```sql
SELECT ROUTINE_NAME
FROM information_schema.ROUTINES
WHERE ROUTINE_SCHEMA = 'shop'
  AND ROUTINE_TYPE   = 'PROCEDURE'
  AND ROUTINE_NAME   = 'get_orders_by_customer';
```

## Permissions Required

The caller needs the `EXECUTE` privilege on the procedure:

```sql
GRANT EXECUTE ON PROCEDURE shop.get_orders_by_customer TO 'app_user'@'%';
```

## Summary

Use the `CALL` statement to invoke a stored procedure in MySQL. For IN parameters, pass literal values or expressions. For OUT and INOUT parameters, pass session variables (prefixed with `@`) so MySQL can write the results back to them. Most client libraries provide a `callproc` or equivalent method that handles result sets and output parameters automatically.
