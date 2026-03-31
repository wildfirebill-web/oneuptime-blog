# How to Use AUTO_INCREMENT in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, AUTO_INCREMENT, Primary Key, Database Design

Description: Learn how to use AUTO_INCREMENT in MySQL to automatically generate unique integer values for primary keys and other sequential identifiers.

---

## What Is AUTO_INCREMENT?

`AUTO_INCREMENT` is a column attribute in MySQL that automatically generates a unique integer value each time a new row is inserted. It is most commonly used for primary key columns to ensure every row has a unique identifier without requiring application-level ID management.

```sql
CREATE TABLE users (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(50) NOT NULL,
    email VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

## Inserting Rows with AUTO_INCREMENT

When inserting rows, omit the AUTO_INCREMENT column or pass `NULL` or `0` and MySQL assigns the next value automatically:

```sql
-- Omit the AUTO_INCREMENT column
INSERT INTO users (username, email) VALUES ('alice', 'alice@example.com');
INSERT INTO users (username, email) VALUES ('bob', 'bob@example.com');

-- Pass NULL explicitly
INSERT INTO users (id, username, email) VALUES (NULL, 'carol', 'carol@example.com');

-- Pass 0 (treated as NULL for AUTO_INCREMENT)
INSERT INTO users (id, username, email) VALUES (0, 'dave', 'dave@example.com');

-- Retrieve the last inserted ID
SELECT LAST_INSERT_ID();
```

## Setting the Starting Value

You can set a custom starting value for the AUTO_INCREMENT counter:

```sql
-- Set starting value at table creation
CREATE TABLE invoices (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    amount DECIMAL(10, 2),
    issued_date DATE
) AUTO_INCREMENT = 1000;

-- Change the counter value on an existing table
ALTER TABLE invoices AUTO_INCREMENT = 5000;
```

## Checking the Current AUTO_INCREMENT Value

```sql
-- View current AUTO_INCREMENT value via INFORMATION_SCHEMA
SELECT AUTO_INCREMENT
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA = 'your_database'
  AND TABLE_NAME = 'users';

-- Alternative using SHOW TABLE STATUS
SHOW TABLE STATUS LIKE 'users'\G
```

## AUTO_INCREMENT with Different Integer Types

Choose the right integer type based on how many rows you expect:

```sql
-- TINYINT UNSIGNED: 0 to 255 rows (not recommended for most tables)
CREATE TABLE small_lookup (
    id TINYINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    code VARCHAR(10)
);

-- INT UNSIGNED: ~4.3 billion rows
CREATE TABLE standard_table (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    data VARCHAR(100)
);

-- BIGINT UNSIGNED: ~18.4 quintillion rows
CREATE TABLE massive_table (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    data TEXT
);
```

## Gaps in AUTO_INCREMENT Sequences

AUTO_INCREMENT values are not reused when rows are deleted or when inserts fail. This is by design for safety:

```sql
-- Delete rows - the IDs are NOT reused
DELETE FROM users WHERE id = 3;

-- The next insert gets id = 5 (not 3)
INSERT INTO users (username, email) VALUES ('eve', 'eve@example.com');

-- To find gaps in an AUTO_INCREMENT sequence
SELECT a.id + 1 AS gap_start
FROM users a
WHERE NOT EXISTS (
    SELECT 1 FROM users b WHERE b.id = a.id + 1
)
ORDER BY a.id;
```

## Resetting AUTO_INCREMENT

After bulk deletions, you may want to reset the counter:

```sql
-- Reset to the minimum safe value (highest existing id + 1)
ALTER TABLE users AUTO_INCREMENT = 1;
-- MySQL automatically adjusts to MAX(id) + 1 if the specified value is too low

-- After TRUNCATE, the counter resets to 1 automatically
TRUNCATE TABLE users;
```

## AUTO_INCREMENT in Multi-Column Primary Keys

AUTO_INCREMENT can be used in composite primary keys in MyISAM or with InnoDB if paired with a non-primary unique key:

```sql
-- InnoDB: AUTO_INCREMENT must be the primary key or part of an index
CREATE TABLE log_entries (
    log_id INT AUTO_INCREMENT,
    server_id INT,
    message TEXT,
    PRIMARY KEY (log_id),
    INDEX idx_server (server_id)
);
```

## Summary

`AUTO_INCREMENT` simplifies unique ID generation in MySQL by automatically assigning the next available integer value on each insert. Use `LAST_INSERT_ID()` to retrieve the generated value after an insert. Choose `INT UNSIGNED` for most tables and `BIGINT UNSIGNED` for very large tables. Remember that gaps in the sequence are normal and expected behavior.
