# How to Prevent SQL Injection Using Prepared Statements in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Security, Prepared Statement, SQL Injection, Query

Description: Learn how parameterized prepared statements eliminate SQL injection vulnerabilities and how to implement them safely in MySQL.

---

## What is SQL Injection?

SQL injection occurs when user-supplied input is concatenated directly into a SQL query. An attacker can manipulate the query structure to bypass authentication, read unauthorized data, modify records, or drop tables.

```sql
-- VULNERABLE: user input concatenated directly into query
SET @email = 'admin@example.com\' OR \'1\'=\'1';
SET @sql = CONCAT('SELECT * FROM users WHERE email = \'', @email, '\'');
-- Results in: SELECT * FROM users WHERE email = 'admin@example.com' OR '1'='1'
-- This returns ALL rows in the users table
```

## How Prepared Statements Prevent Injection

When you use `PREPARE` with `?` placeholders, MySQL separates the query structure from the data. The query is parsed and compiled with the placeholders. When you `EXECUTE` with `USING`, the parameter values are bound after parsing - they can never alter the query structure.

```sql
-- SAFE: parameterized query
PREPARE get_user FROM 'SELECT * FROM users WHERE email = ?';
SET @email = 'admin@example.com\' OR \'1\'=\'1';
EXECUTE get_user USING @email;
-- MySQL treats the entire string as a literal value, not SQL syntax
-- Returns 0 rows (the email does not exist)
DEALLOCATE PREPARE get_user;
```

## Server-Side vs Client-Side Prepared Statements

MySQL supports both:

- **Server-side** (protocol-level): Used by driver libraries like `mysqli`, `PDO`, and `mysql2`. The `?` placeholder is sent to the server for compilation before any data.
- **Client-side** (emulated): Some drivers emulate prepared statements by escaping and interpolating parameters client-side. This is less secure if escaping is bypassed.

Always prefer server-side prepared statements. In PHP PDO:

```php
// Disable emulated prepared statements (use real server-side)
$pdo->setAttribute(PDO::ATTR_EMULATE_PREPARES, false);
$stmt = $pdo->prepare('SELECT * FROM users WHERE email = ?');
$stmt->execute([$email]);
```

## Examples in Common Languages

### Python (mysql-connector-python)

```python
import mysql.connector

conn = mysql.connector.connect(host='localhost', user='app', password='pass', database='mydb')
cursor = conn.cursor()

# Safe parameterized query
query = "SELECT id, name FROM users WHERE email = %s"
cursor.execute(query, (user_input_email,))
results = cursor.fetchall()
```

### Node.js (mysql2)

```javascript
const mysql = require('mysql2/promise');
const conn = await mysql.createConnection({ host: 'localhost', user: 'app', password: 'pass', database: 'mydb' });

// Safe parameterized query
const [rows] = await conn.execute(
  'SELECT id, name FROM users WHERE email = ?',
  [userInputEmail]
);
```

### Java (JDBC)

```java
PreparedStatement stmt = conn.prepareStatement(
    "SELECT id, name FROM users WHERE email = ?"
);
stmt.setString(1, userInputEmail);
ResultSet rs = stmt.executeQuery();
```

## What Prepared Statements Cannot Protect

Prepared statements protect data values only. They cannot parameterize structural SQL elements. If you dynamically build table names or column names from user input, you are still vulnerable:

```sql
-- STILL VULNERABLE: table name from user input
SET @sql = CONCAT('SELECT * FROM ', user_supplied_table);
PREPARE stmt FROM @sql;
```

For structural elements, use a strict allowlist:

```sql
IF user_supplied_table NOT IN ('orders', 'products', 'customers') THEN
  SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Invalid table name';
END IF;
SET @sql = CONCAT('SELECT * FROM ', user_supplied_table);
```

## Summary

Prepared statements with `?` placeholders prevent SQL injection by separating query structure from data. Parameters are bound after the query is parsed, so user input can never change the query logic. Use server-side prepared statements in your application driver, and apply allowlist validation for any structural SQL elements like table or column names that must be dynamic.
