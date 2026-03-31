# How to Convert Data Types in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Data Type, Conversion, SQL, Database

Description: Learn how to convert data types in MySQL using CAST, CONVERT, implicit coercion, and ALTER TABLE to change column types permanently.

---

## Overview of Type Conversion in MySQL

MySQL supports two kinds of type conversion:

1. **Explicit conversion** - You tell MySQL exactly what type to convert to using `CAST()` or `CONVERT()`
2. **Implicit conversion** - MySQL automatically converts types when comparing or operating on mismatched types

Understanding both is essential for writing correct and efficient queries.

## Explicit Conversion with CAST()

```sql
CAST(expression AS type)
```

Supported target types: `BINARY`, `CHAR`, `DATE`, `DATETIME`, `DECIMAL`, `DOUBLE`, `FLOAT`, `JSON`, `REAL`, `SIGNED`, `TIME`, `UNSIGNED`

```sql
-- String to integer
SELECT CAST('42' AS SIGNED);          -- 42

-- String to decimal
SELECT CAST('3.14' AS DECIMAL(10,2)); -- 3.14

-- Integer to string
SELECT CAST(100 AS CHAR);             -- '100'

-- String to date
SELECT CAST('2026-03-31' AS DATE);    -- 2026-03-31

-- String to datetime
SELECT CAST('2026-03-31 12:00:00' AS DATETIME);
```

## Explicit Conversion with CONVERT()

`CONVERT()` works similarly to `CAST()` but also supports character set conversion:

```sql
-- Type conversion (same as CAST)
SELECT CONVERT('42', SIGNED);
SELECT CONVERT('2026-03-31', DATE);

-- Character set conversion
SELECT CONVERT('hello' USING utf8mb4);
SELECT CONVERT(name USING latin1) FROM users;
```

## Converting Numbers to Strings

```sql
SELECT CAST(price AS CHAR) FROM products;
-- or
SELECT price + 0.0 FROM products;    -- implicit decimal conversion
SELECT CONCAT(price, '') FROM products; -- forces string context
```

## Converting Strings to Numbers

```sql
SELECT CAST('123abc' AS SIGNED);  -- Returns 123 (truncates at first non-numeric char)
SELECT CAST('abc' AS SIGNED);     -- Returns 0

-- Check if a string is purely numeric
SELECT * FROM products WHERE CAST(sku AS SIGNED) > 0 AND sku REGEXP '^[0-9]+$';
```

## Converting Between Date and String

```sql
-- Date to string
SELECT DATE_FORMAT(NOW(), '%Y-%m-%d');           -- '2026-03-31'
SELECT CAST(NOW() AS CHAR);                      -- '2026-03-31 12:00:00'

-- String to date
SELECT STR_TO_DATE('31/03/2026', '%d/%m/%Y');    -- 2026-03-31
SELECT CAST('2026-03-31' AS DATE);               -- 2026-03-31
```

## Implicit Type Conversion

MySQL automatically converts types in comparisons:

```sql
-- String compared to integer: string is converted to integer
SELECT * FROM orders WHERE id = '42';   -- Works, id is INT

-- Warning: comparing indexed INT column with a string bypasses the index
SELECT * FROM orders WHERE CAST(id AS CHAR) = '42';  -- Avoid this
```

## Permanently Changing a Column Type with ALTER TABLE

To convert a column's data type in the schema itself:

```sql
-- Change VARCHAR to INT
ALTER TABLE products MODIFY COLUMN price INT;

-- Change INT to DECIMAL
ALTER TABLE products MODIFY COLUMN price DECIMAL(10, 2);

-- Change VARCHAR to TEXT
ALTER TABLE articles MODIFY COLUMN body TEXT;
```

MySQL will attempt to convert existing data. Check for warnings:

```sql
ALTER TABLE products MODIFY COLUMN price INT;
SHOW WARNINGS;
```

## Handling Conversion Failures

When a string cannot be converted, MySQL returns 0 (for numbers) or NULL (in strict mode):

```sql
-- Strict mode OFF: returns 0 with a warning
SELECT CAST('not-a-number' AS SIGNED);

-- Check strict mode
SHOW VARIABLES LIKE 'sql_mode';
```

Enable strict mode to get errors instead of silent truncation:

```sql
SET sql_mode = 'STRICT_TRANS_TABLES';
```

## Summary

MySQL provides `CAST()` and `CONVERT()` for explicit type conversion in queries. Use `CAST(expr AS type)` for most type changes, `CONVERT(expr USING charset)` for character set changes, and `ALTER TABLE ... MODIFY COLUMN` to permanently change a column's type. Be aware of implicit conversions in `WHERE` clauses, as they can silently bypass indexes and return unexpected results.
