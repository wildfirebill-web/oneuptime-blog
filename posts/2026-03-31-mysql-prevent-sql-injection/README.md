# How to Prevent SQL Injection in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Security, SQL Injection, Prepared Statement, Best Practice

Description: Learn how to prevent SQL injection attacks in MySQL applications using prepared statements, input validation, and least-privilege database design.

---

SQL injection is one of the most critical and widespread web application vulnerabilities. It occurs when untrusted user input is embedded directly into SQL queries, allowing attackers to manipulate query logic, bypass authentication, dump data, or delete records. MySQL applications are not inherently protected - prevention requires deliberate coding practices.

## Understanding SQL Injection

A vulnerable query concatenates user input directly:

```python
# VULNERABLE - Never do this
username = request.args.get('username')
query = "SELECT * FROM users WHERE username = '" + username + "'"
cursor.execute(query)
```

An attacker can submit `' OR '1'='1` as the username to bypass authentication or `'; DROP TABLE users; --` to destroy data.

## Using Prepared Statements (Primary Defense)

Prepared statements separate SQL logic from data values, making injection impossible:

```python
# SAFE - Using prepared statements
username = request.args.get('username')
cursor.execute("SELECT * FROM users WHERE username = %s", (username,))
```

```php
// SAFE - PHP PDO prepared statement
$stmt = $pdo->prepare("SELECT * FROM users WHERE username = ?");
$stmt->execute([$username]);
$user = $stmt->fetch();
```

```java
// SAFE - Java PreparedStatement
String sql = "SELECT * FROM users WHERE username = ?";
PreparedStatement stmt = conn.prepareStatement(sql);
stmt.setString(1, username);
ResultSet rs = stmt.executeQuery();
```

## Using Parameterized Queries in Node.js

```javascript
const mysql = require('mysql2/promise');

async function getUserByUsername(username) {
  const [rows] = await connection.execute(
    'SELECT id, email FROM users WHERE username = ?',
    [username]
  );
  return rows[0];
}
```

## Input Validation as Defense in Depth

While not a replacement for prepared statements, validate and sanitize input:

```python
import re

def validate_username(username):
    # Allow only alphanumeric and underscores
    if not re.match(r'^[a-zA-Z0-9_]{3,50}$', username):
        raise ValueError("Invalid username format")
    return username
```

## Using an ORM

ORMs like SQLAlchemy or Sequelize use parameterized queries internally:

```python
# SQLAlchemy - safe by default
from sqlalchemy.orm import Session

user = session.query(User).filter(User.username == username).first()
```

## Principle of Least Privilege

Even if injection occurs, limit damage by restricting database permissions:

```sql
-- Create a read-only user for the API
CREATE USER 'api_readonly'@'app-server'
  IDENTIFIED BY 'StrongPassword!';
GRANT SELECT ON mydb.users TO 'api_readonly'@'app-server';

-- Separate write user with minimal access
CREATE USER 'api_writer'@'app-server'
  IDENTIFIED BY 'DifferentPassword!';
GRANT SELECT, INSERT ON mydb.orders TO 'api_writer'@'app-server';
```

## Using MySQL's built-in Escaping as Last Resort

If prepared statements are not possible, escape manually:

```python
import mysql.connector

username = mysql.connector.conversion.MySQLConverter().escape(user_input)
# Then use in query - but prepared statements are always preferable
```

## Enable the MySQL General Log for Auditing

```sql
SET GLOBAL general_log = ON;
SET GLOBAL general_log_file = '/var/log/mysql/general.log';
```

Monitor for suspicious patterns like multiple statements or comment sequences (`--`, `/*`).

## Summary

The primary defense against SQL injection in MySQL is using prepared statements with parameterized queries. Every other practice - input validation, ORMs, least-privilege accounts, and logging - adds defense in depth but does not replace parameterized queries. Never concatenate user input into SQL strings, and enforce this standard in code reviews.
