# What Is a Prepared Statement in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Prepared Statement, Security, SQL Injection, Performance

Description: A prepared statement in MySQL is a precompiled SQL template with placeholders that separates query structure from data, preventing SQL injection and enabling query reuse.

---

## Overview

A prepared statement is a SQL statement that is parsed and compiled once by MySQL, then executed multiple times with different parameter values. The statement contains placeholder markers (`?`) where data values will be supplied at execution time. This separation of SQL structure from data is the primary defense against SQL injection attacks. It also offers a performance benefit when the same statement runs many times, since the parsing and optimization step is done only once.

## Server-Side Prepared Statements (SQL Syntax)

MySQL's `PREPARE`, `EXECUTE`, and `DEALLOCATE PREPARE` commands work directly in SQL:

```sql
-- Prepare the statement
PREPARE get_user FROM 'SELECT id, name, email FROM users WHERE id = ?';

-- Execute with a value
SET @user_id = 42;
EXECUTE get_user USING @user_id;

-- Execute again with a different value
SET @user_id = 99;
EXECUTE get_user USING @user_id;

-- Release the prepared statement
DEALLOCATE PREPARE get_user;
```

## Parameterized Queries in Application Code

Most MySQL connectors use the binary protocol for prepared statements, which is faster and more secure than string interpolation:

```python
import mysql.connector

conn = mysql.connector.connect(host='localhost', user='app', password='secret', database='mydb')
cursor = conn.cursor(prepared=True)

# The query is prepared once
cursor.execute(
    "SELECT id, name FROM products WHERE category_id = %s AND price < %s",
    (5, 100.00)
)
rows = cursor.fetchall()
```

```javascript
const mysql = require('mysql2/promise');
const conn = await mysql.createConnection({host: 'localhost', user: 'app', password: 'secret'});

// Parameterized query -- prevents SQL injection
const [rows] = await conn.execute(
  'SELECT * FROM orders WHERE customer_id = ? AND status = ?',
  [customerId, 'pending']
);
```

## SQL Injection Prevention

Without prepared statements, string concatenation creates injection vulnerabilities:

```php
// UNSAFE: SQL injection possible
$query = "SELECT * FROM users WHERE name = '" . $_GET['name'] . "'";

// SAFE: prepared statement with parameter binding
$stmt = $pdo->prepare("SELECT * FROM users WHERE name = ?");
$stmt->execute([$_GET['name']]);
```

If a user submits `' OR '1'='1`, the prepared statement treats it as a literal string, not SQL syntax.

## Dynamic SQL with Prepared Statements

Prepared statements also enable dynamic query construction inside stored procedures:

```sql
DELIMITER $$

CREATE PROCEDURE search_orders(IN p_status VARCHAR(50))
BEGIN
  SET @sql = CONCAT('SELECT * FROM orders WHERE status = ?');
  SET @status_val = p_status;
  PREPARE stmt FROM @sql;
  EXECUTE stmt USING @status_val;
  DEALLOCATE PREPARE stmt;
END$$

DELIMITER ;
```

## Limitations

- Prepared statement parameters cannot be used for identifiers (table names, column names). You must use string concatenation for dynamic identifiers and sanitize those manually.
- In the SQL interface, only session-level user variables (`@var`) can be used as parameters with `USING`.
- Prepared statements are session-scoped and not shared between connections.

```sql
-- This does NOT work -- column name as parameter
PREPARE stmt FROM 'SELECT ? FROM orders';

-- Use dynamic SQL carefully for identifiers
SET @col = 'customer_id';
SET @sql = CONCAT('SELECT ', @col, ' FROM orders LIMIT 10');
PREPARE stmt FROM @sql;
EXECUTE stmt;
DEALLOCATE PREPARE stmt;
```

## Summary

Prepared statements in MySQL separate SQL structure from data using placeholders, preventing SQL injection by ensuring user-supplied values are never interpreted as SQL code. They also enable query reuse with a single parse-and-optimize step. All production MySQL applications should use parameterized queries through their connector library rather than building SQL strings by concatenation.
