# What Is an AUTO_INCREMENT Column in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Auto Increment, Primary Key, Database Design

Description: An AUTO_INCREMENT column in MySQL automatically generates a unique sequential integer value for each new row, commonly used as a surrogate primary key.

---

## Overview

`AUTO_INCREMENT` is a column attribute in MySQL that causes the server to automatically generate a unique, incrementing integer value when a new row is inserted without explicitly providing a value for that column. It is most commonly applied to primary key columns to act as a surrogate identifier.

## Creating a Table with AUTO_INCREMENT

```sql
CREATE TABLE users (
  id INT UNSIGNED NOT NULL AUTO_INCREMENT,
  email VARCHAR(255) NOT NULL UNIQUE,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (id)
);
```

## Inserting Rows

```sql
-- No need to specify id - it's auto-generated
INSERT INTO users (email) VALUES ('alice@example.com');
INSERT INTO users (email) VALUES ('bob@example.com');

SELECT id, email FROM users;
-- +----+-------------------+
-- | id | email             |
-- +----+-------------------+
-- |  1 | alice@example.com |
-- |  2 | bob@example.com   |
```

## Getting the Last Inserted ID

```sql
INSERT INTO users (email) VALUES ('carol@example.com');
SELECT LAST_INSERT_ID();
-- Returns 3
```

In application code:

```python
cursor.execute("INSERT INTO users (email) VALUES (%s)", ('dave@example.com',))
new_id = cursor.lastrowid
print(f"Inserted with ID: {new_id}")
```

## Checking the Current AUTO_INCREMENT Value

```sql
-- Via INFORMATION_SCHEMA
SELECT auto_increment
FROM information_schema.tables
WHERE table_schema = 'mydb' AND table_name = 'users';

-- Via SHOW TABLE STATUS
SHOW TABLE STATUS FROM mydb LIKE 'users'\G
```

## Setting a Custom Starting Value

```sql
-- Start IDs at 1000
CREATE TABLE orders (
  id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  total DECIMAL(10,2)
) AUTO_INCREMENT = 1000;

-- Or alter an existing table
ALTER TABLE orders AUTO_INCREMENT = 5000;
```

## Resetting AUTO_INCREMENT

```sql
-- Reset to the next available value after existing max
ALTER TABLE orders AUTO_INCREMENT = 1;

-- MySQL will set it to max(id) + 1, not literally 1 if rows exist
```

## Using BIGINT for Large Tables

```sql
-- Use BIGINT UNSIGNED for tables that will grow very large
CREATE TABLE events (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
  event_type VARCHAR(50),
  created_at DATETIME
);
```

`INT UNSIGNED` allows up to ~4.29 billion rows. `BIGINT UNSIGNED` allows ~18.4 quintillion.

## AUTO_INCREMENT Lock Modes

InnoDB has three lock modes controlling concurrency during AUTO_INCREMENT inserts:

```sql
SHOW VARIABLES LIKE 'innodb_autoinc_lock_mode';
```

| Mode | Value | Behavior |
|------|-------|----------|
| Traditional | 0 | Table-level lock for all inserts |
| Consecutive | 1 | Statement-level lock for bulk inserts (default pre-8.0) |
| Interleaved | 2 | No locks - concurrent inserts can have gaps (default MySQL 8.0) |

## Gaps in AUTO_INCREMENT Sequences

AUTO_INCREMENT values are not guaranteed to be gap-free. Gaps occur when:
- Rows are deleted
- Transactions are rolled back
- `INSERT IGNORE` silently fails
- Bulk insert statements pre-allocate IDs

Do not rely on AUTO_INCREMENT for sequential gap-free numbering.

## Composite Primary Keys with AUTO_INCREMENT

With MyISAM, AUTO_INCREMENT can be used on non-primary columns in a composite key:

```sql
-- MyISAM only
CREATE TABLE logs (
  server_id INT,
  log_id INT AUTO_INCREMENT,
  message TEXT,
  PRIMARY KEY (server_id, log_id)
) ENGINE = MyISAM;
```

InnoDB requires AUTO_INCREMENT columns to be indexed.

## Summary

`AUTO_INCREMENT` is MySQL's built-in mechanism for generating unique sequential identifiers for rows. It works seamlessly with integer primary keys, provides `LAST_INSERT_ID()` for retrieving generated values in code, and supports customizable starting values. Use `BIGINT UNSIGNED` for large tables, and be aware that gaps in the sequence are normal and expected behavior.
