# How to Use DATABASE() Function in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, DATABASE Function, Information Functions, SQL

Description: Learn how to use the DATABASE() function in MySQL to retrieve the name of the current default database in queries and stored procedures.

---

## What is DATABASE()

MySQL's `DATABASE()` function returns the name of the current default database (the one selected with `USE`). If no database is selected, it returns NULL.

Syntax:

```sql
DATABASE()
-- or equivalently:
SCHEMA()
```

Both `DATABASE()` and `SCHEMA()` are synonyms in MySQL.

## Basic Usage

```sql
-- Check current database
SELECT DATABASE();

-- If connected to 'ecommerce' database:
-- Output: ecommerce
```

If no database is selected:

```sql
SELECT DATABASE();
-- Output: NULL
```

## Selecting a Database and Verifying

```sql
USE myapp;
SELECT DATABASE();
-- Output: myapp

USE information_schema;
SELECT DATABASE();
-- Output: information_schema
```

## Using DATABASE() in Queries

`DATABASE()` is useful when querying the `information_schema` to filter to the current database:

```sql
-- Get all tables in the current database
SELECT table_name, engine, table_rows
FROM information_schema.tables
WHERE table_schema = DATABASE()
ORDER BY table_name;
```

```sql
-- Get all columns for all tables in current database
SELECT table_name, column_name, data_type
FROM information_schema.columns
WHERE table_schema = DATABASE()
ORDER BY table_name, ordinal_position;
```

```sql
-- Get all indexes in the current database
SELECT table_name, index_name, column_name, non_unique
FROM information_schema.statistics
WHERE table_schema = DATABASE()
ORDER BY table_name, index_name;
```

## DATABASE() in Stored Procedures

```sql
DELIMITER //
CREATE PROCEDURE show_current_db()
BEGIN
  SELECT DATABASE() AS current_database;
END //
DELIMITER ;

CALL show_current_db();
```

Useful for debugging which database a procedure is running against:

```sql
DELIMITER //
CREATE PROCEDURE validate_schema(IN expected_db VARCHAR(100))
BEGIN
  IF DATABASE() != expected_db THEN
    SIGNAL SQLSTATE '45000'
    SET MESSAGE_TEXT = 'Wrong database - operation aborted';
  END IF;
  -- rest of procedure
END //
DELIMITER ;
```

## DATABASE() vs SELECT FROM information_schema

```sql
-- These are equivalent
SELECT DATABASE();

SELECT schema_name
FROM information_schema.schemata
WHERE schema_name = DATABASE();
```

## Using DATABASE() in Dynamic SQL

```sql
-- Build dynamic table check
SET @db = DATABASE();
SET @sql = CONCAT('SELECT COUNT(*) FROM information_schema.tables WHERE table_schema = ''', @db, '''');
PREPARE stmt FROM @sql;
EXECUTE stmt;
DEALLOCATE PREPARE stmt;
```

## Checking Database in Application Code

In Python:

```python
cursor.execute("SELECT DATABASE()")
current_db = cursor.fetchone()[0]
print(f"Connected to: {current_db}")
```

In Node.js:

```javascript
const [rows] = await conn.execute('SELECT DATABASE() AS db');
console.log('Current database:', rows[0].db);
```

## Practical Use Case - Audit Logging

```sql
DELIMITER //
CREATE TRIGGER log_employee_changes
AFTER UPDATE ON employees
FOR EACH ROW
BEGIN
  INSERT INTO audit_log (table_name, database_name, changed_at, changed_by)
  VALUES ('employees', DATABASE(), NOW(), USER());
END //
DELIMITER ;
```

## Summary

`DATABASE()` (or `SCHEMA()`) in MySQL returns the currently selected database name. It is most useful when writing portable queries against `information_schema`, storing context in audit tables, and validating that stored procedures are running against the expected database. Returns NULL if no database is selected.
