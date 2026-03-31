# What Is STRICT_TRANS_TABLES Mode in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, SQL Mode, STRICT_TRANS_TABLES, Data Validation, InnoDB

Description: STRICT_TRANS_TABLES is a MySQL SQL mode that rejects invalid or out-of-range data values for transactional tables, preventing silent data truncation and corruption.

---

## Overview

`STRICT_TRANS_TABLES` is a SQL mode flag in MySQL that enables strict data validation for transactional storage engines (primarily InnoDB). When active, MySQL raises an error and rolls back the statement if you attempt to insert or update a value that is invalid for a column's data type or exceeds the column's defined length. Without this mode, MySQL silently adjusts (truncates or coerces) bad data and proceeds, which can lead to subtle data corruption.

This mode is enabled by default in MySQL 5.7 and later.

## The Problem Without STRICT Mode

Without `STRICT_TRANS_TABLES`, MySQL silently adjusts invalid data:

```sql
-- Disable strict mode
SET SESSION sql_mode = '';

CREATE TABLE products (
  id INT PRIMARY KEY,
  name VARCHAR(10),
  price DECIMAL(5,2)
);

-- Name is 20 chars, price exceeds DECIMAL(5,2) range
INSERT INTO products (id, name, price) VALUES (1, 'VeryLongProductName', 9999.999);

-- No error! MySQL silently inserts: id=1, name='VeryLongPr', price=999.99
SELECT * FROM products;
```

The application receives no error but the data is silently corrupted.

## With STRICT_TRANS_TABLES Enabled

```sql
SET SESSION sql_mode = 'STRICT_TRANS_TABLES';

INSERT INTO products (id, name, price) VALUES (1, 'VeryLongProductName', 9999.999);
-- ERROR 1406 (22001): Data too long for column 'name'

INSERT INTO products (id, name, price) VALUES (2, 'Widget', 9999.999);
-- ERROR 1264 (22003): Out of range value for column 'price'
```

The transaction is rolled back and the data is not inserted.

## What STRICT_TRANS_TABLES Covers

The mode enforces data integrity for:
- String values that exceed column length (`VARCHAR`, `CHAR`, `TEXT`)
- Numeric values outside the column range (`INT`, `DECIMAL`, `FLOAT`)
- Invalid values for `ENUM` columns
- NULL values in NOT NULL columns (without a DEFAULT)
- Invalid date/time values (in combination with `NO_ZERO_DATE`)

```sql
SET SESSION sql_mode = 'STRICT_TRANS_TABLES';

CREATE TABLE orders (
  id INT AUTO_INCREMENT PRIMARY KEY,
  status ENUM('pending', 'shipped', 'cancelled') NOT NULL
);

INSERT INTO orders (status) VALUES ('unknown_status');
-- ERROR 1265 (01000): Data truncated for column 'status'
```

## STRICT_TRANS_TABLES vs STRICT_ALL_TABLES

- `STRICT_TRANS_TABLES`: Strict only for transactional tables. For non-transactional tables, adjustments are still silently made for multi-row inserts if the first row succeeds.
- `STRICT_ALL_TABLES`: Strict for both transactional and non-transactional tables. Raises an error on any invalid data even for non-transactional engines.

Most production systems use `STRICT_TRANS_TABLES` since InnoDB is the standard engine.

## Checking Whether Strict Mode Is Active

```sql
SELECT @@sql_mode LIKE '%STRICT_TRANS_TABLES%';
-- Returns 1 if enabled, 0 if not
```

## Application Compatibility

Enabling `STRICT_TRANS_TABLES` on a system previously running without it may surface hidden data issues:

```bash
# Test with strict mode before enabling globally
mysql -e "SET SESSION sql_mode='STRICT_TRANS_TABLES'; source /path/to/app_queries.sql" 2>&1 | grep ERROR
```

Fix any errors that surface before enabling the mode in production.

## Summary

`STRICT_TRANS_TABLES` is a critical MySQL SQL mode that rejects invalid data instead of silently adjusting it. It protects against data truncation, out-of-range values, and invalid ENUM entries for InnoDB tables. Enabled by default in MySQL 5.7+, it should always be active in production systems to prevent subtle data corruption that manifests as incorrect application behavior rather than explicit database errors.
