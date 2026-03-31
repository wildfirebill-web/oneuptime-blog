# What Is a Prepared Statement in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Prepared Statements, SQL Injection, Security, Performance, Protocol

Description: Learn what a MySQL prepared statement is, how it prevents SQL injection, improves performance for repeated queries, and how to use it in SQL and application code.

---

## What Is a Prepared Statement

A prepared statement is a pre-compiled SQL template that can be executed multiple times with different parameter values. The SQL structure is sent to MySQL once and parsed once. Subsequent executions supply only the parameter values, not the full SQL text.

Prepared statements provide two major benefits:
1. Security - parameters are always treated as data, never as SQL syntax, preventing SQL injection
2. Performance - the query is parsed and optimized once, then executed multiple times efficiently

## How Prepared Statements Work

The lifecycle of a prepared statement has three steps:

1. **PREPARE** - send the SQL template to MySQL; MySQL parses it and returns a statement handle
2. **EXECUTE** - bind parameter values and execute the prepared statement
3. **DEALLOCATE** - release the prepared statement when done

## Using Prepared Statements in SQL

MySQL supports prepared statements directly in SQL using the `PREPARE`, `EXECUTE`, and `DEALLOCATE PREPARE` commands:

```sql
-- Step 1: Prepare the statement
PREPARE get_user FROM 'SELECT id, name, email FROM users WHERE id = ?';

-- Step 2: Set the parameter and execute
SET @user_id = 42;
EXECUTE get_user USING @user_id;

-- Execute again with a different value
SET @user_id = 101;
EXECUTE get_user USING @user_id;

-- Step 3: Release the prepared statement
DEALLOCATE PREPARE get_user;
```

The `?` placeholder marks the position where parameter values will be substituted.

## Multi-Parameter Example

```sql
PREPARE insert_order FROM
  'INSERT INTO orders (customer_id, product_id, quantity, total)
   VALUES (?, ?, ?, ?)';

SET @cid = 5;
SET @pid = 12;
SET @qty = 3;
SET @total = 149.97;

EXECUTE insert_order USING @cid, @pid, @qty, @total;

-- Execute again for another order
SET @cid = 7;
SET @pid = 8;
SET @qty = 1;
SET @total = 49.99;

EXECUTE insert_order USING @cid, @pid, @qty, @total;

DEALLOCATE PREPARE insert_order;
```

## Using Prepared Statements from Application Code

Most MySQL client libraries support prepared statements through their native API. Here is an example in Python using `mysql-connector-python`:

```python
import mysql.connector

conn = mysql.connector.connect(
    host='localhost',
    user='app_user',
    password='secret',
    database='myapp'
)
cursor = conn.cursor(prepared=True)

# Prepare once
sql = "SELECT id, name, email FROM users WHERE email = %s AND active = %s"
cursor.execute(sql, ('alice@example.com', 1))
row = cursor.fetchone()
print(row)

# Reuse with different parameters
cursor.execute(sql, ('bob@example.com', 1))
row = cursor.fetchone()
print(row)

cursor.close()
conn.close()
```

In PHP with PDO:

```php
$pdo = new PDO('mysql:host=localhost;dbname=myapp', 'user', 'secret');

$stmt = $pdo->prepare('SELECT id, name FROM users WHERE id = :id');
$stmt->execute(['id' => 42]);
$user = $stmt->fetch(PDO::FETCH_ASSOC);

// Reuse
$stmt->execute(['id' => 101]);
$user = $stmt->fetch(PDO::FETCH_ASSOC);
```

## SQL Injection Prevention

Without prepared statements, concatenating user input directly into SQL creates injection vulnerabilities:

```python
# DANGEROUS - never do this
user_input = "1 OR 1=1"
query = f"SELECT * FROM users WHERE id = {user_input}"
# Executes: SELECT * FROM users WHERE id = 1 OR 1=1
# Returns all users!
```

With prepared statements, the parameter value is always treated as data:

```python
# SAFE - parameter value cannot change the SQL structure
cursor.execute("SELECT * FROM users WHERE id = %s", (user_input,))
# MySQL treats user_input as a literal string value, not SQL
```

## Performance Considerations

Prepared statements benefit repeated queries by:
- Reducing parsing overhead (parsed once, executed many times)
- Reducing network overhead (only parameter values are sent on re-execution)

For queries executed only once, the prepare and deallocate round trips may add more overhead than a regular query. Use prepared statements when the same query pattern executes many times, such as in loops or high-traffic application code paths.

## Monitoring Prepared Statements

View prepared statement statistics in the Performance Schema:

```sql
SELECT *
FROM performance_schema.prepared_statements_instances
LIMIT 10;
```

Check global prepared statement counters:

```sql
SHOW STATUS LIKE 'Prepared_stmt_count';
SHOW STATUS LIKE 'Com_stmt_%';
```

## Summary

A MySQL prepared statement is a parameterized SQL template that is compiled once and executed multiple times with different values. It is the most effective defense against SQL injection because parameter values are always treated as data by the MySQL protocol, never as SQL syntax. Prepared statements also improve performance for repeated queries by eliminating redundant parsing. Use them consistently in application code wherever user-supplied values are included in SQL queries.
