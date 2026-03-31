# How to Use ENUM Data Type in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Data Types, ENUM, Database Design

Description: Learn how to use the ENUM data type in MySQL to restrict column values to a predefined list and improve data integrity and storage efficiency.

---

## What Is ENUM?

`ENUM` is a string data type that restricts a column to one value from a predefined list. MySQL stores the value as a numeric index internally (1, 2, 3, ...) but displays it as the corresponding string. This makes it very compact - values with up to 255 members use 1 byte, up to 65535 use 2 bytes.

```sql
CREATE TABLE orders (
    id INT AUTO_INCREMENT PRIMARY KEY,
    customer_name VARCHAR(100),
    status ENUM('pending', 'processing', 'shipped', 'delivered', 'cancelled') NOT NULL DEFAULT 'pending',
    priority ENUM('low', 'medium', 'high') DEFAULT 'medium'
);
```

## Inserting ENUM Values

Insert by string value or by numeric index position (1-based):

```sql
-- Insert using string values (recommended for clarity)
INSERT INTO orders (customer_name, status, priority)
VALUES ('Alice Johnson', 'pending', 'high');

INSERT INTO orders (customer_name, status, priority)
VALUES ('Bob Smith', 'processing', 'low');

-- Insert using numeric index (1 = 'pending', 2 = 'processing', etc.)
INSERT INTO orders (customer_name, status) VALUES ('Carol White', 2);
```

## Querying ENUM Columns

```sql
-- Filter by string value
SELECT * FROM orders WHERE status = 'pending';

-- Filter by numeric index
SELECT * FROM orders WHERE status = 1;  -- 1 = 'pending'

-- Sort by ENUM (sorts by internal numeric order, not alphabetically)
SELECT customer_name, status FROM orders ORDER BY status;

-- Sort alphabetically instead
SELECT customer_name, status FROM orders ORDER BY CAST(status AS CHAR);
```

## ENUM Ordering Behavior

ENUM sorts by the internal index, not alphabetically. This can be used intentionally to define a natural order:

```sql
CREATE TABLE tickets (
    id INT AUTO_INCREMENT PRIMARY KEY,
    title VARCHAR(200),
    -- Define in logical workflow order so ORDER BY status is meaningful
    status ENUM('open', 'in_progress', 'review', 'closed') DEFAULT 'open'
);

-- This sorts in workflow order: open -> in_progress -> review -> closed
SELECT * FROM tickets ORDER BY status;
```

## Modifying ENUM Lists

You can add values to an ENUM list with `ALTER TABLE`. Note that inserting values in the middle (rather than appending) causes a table rebuild:

```sql
-- Add a new value at the end (fast, no table rebuild in MySQL 8.0+)
ALTER TABLE orders
MODIFY COLUMN status ENUM('pending', 'processing', 'shipped', 'delivered', 'cancelled', 'refunded');

-- Rename or reorder requires table rebuild
ALTER TABLE orders
MODIFY COLUMN status ENUM('new', 'pending', 'processing', 'shipped', 'delivered', 'cancelled', 'refunded');
```

## Handling Invalid Values

If you try to insert a value not in the ENUM list, MySQL behavior depends on the SQL mode:

```sql
-- With strict mode (default in MySQL 8.0): error is raised
INSERT INTO orders (customer_name, status) VALUES ('Dave', 'unknown');
-- ERROR 1265 (01000): Data truncated for column 'status'

-- Check valid ENUM values using INFORMATION_SCHEMA
SELECT COLUMN_TYPE
FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_NAME = 'orders' AND COLUMN_NAME = 'status';
```

## ENUM vs VARCHAR vs TINYINT

```sql
-- ENUM: self-documenting, compact, validated at DB level
CREATE TABLE products (
    status ENUM('active', 'inactive', 'discontinued') NOT NULL
);

-- VARCHAR: flexible but no built-in validation
CREATE TABLE products_varchar (
    status VARCHAR(20) NOT NULL  -- Any value can be inserted
);

-- TINYINT with application-level mapping: compact but not self-documenting
CREATE TABLE products_int (
    status TINYINT NOT NULL  -- 0=active, 1=inactive, 2=discontinued (needs docs)
);
```

ENUM is the best choice when the list of values is small, stable, and well-known.

## Summary

The ENUM data type is a powerful tool for enforcing data integrity at the database level. It restricts column values to a predefined list, stores them compactly as integers, and sorts them in the order they are defined. Use ENUM for fields like status, priority, category, and type when the allowed values are known and unlikely to change frequently.
