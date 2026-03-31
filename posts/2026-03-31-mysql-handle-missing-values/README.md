# How to Handle Missing Values in MySQL Data

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Data Cleaning, NULL, COALESCE, Default

Description: Learn how to find, fill, and prevent missing values in MySQL using COALESCE, IFNULL, DEFAULT constraints, and conditional update strategies.

---

## Types of Missing Values in MySQL

MySQL represents missing data as `NULL`. However, missing data can also appear as empty strings `''`, zero values `0`, or placeholder strings like `'N/A'`. Each requires a different treatment strategy.

## Finding NULL Values

Count NULLs per column to understand the scope of the problem:

```sql
SELECT
  SUM(CASE WHEN phone IS NULL THEN 1 ELSE 0 END)   AS null_phone,
  SUM(CASE WHEN address IS NULL THEN 1 ELSE 0 END)  AS null_address,
  SUM(CASE WHEN city IS NULL THEN 1 ELSE 0 END)     AS null_city,
  COUNT(*) AS total_rows
FROM customers;
```

Find rows missing critical fields:

```sql
SELECT id, name, email
FROM customers
WHERE phone IS NULL
   OR address IS NULL;
```

## Finding Empty Strings and Placeholders

```sql
-- Find empty strings
SELECT id, name FROM customers WHERE phone = '';

-- Find placeholder values
SELECT id, country FROM customers WHERE country IN ('N/A', 'unknown', 'none', '-');

-- Normalize them to NULL
UPDATE customers SET phone   = NULL WHERE phone = '';
UPDATE customers SET country = NULL WHERE country IN ('N/A', 'unknown', 'none', '-');
```

## Filling Missing Values with COALESCE

Use `COALESCE` to substitute a default at query time without modifying stored data:

```sql
SELECT
  id,
  name,
  COALESCE(phone, 'Not provided') AS phone,
  COALESCE(city, state, country, 'Unknown location') AS location
FROM customers;
```

`COALESCE` returns the first non-NULL value from the list.

## Updating Missing Values with Defaults

For data that should have a known default, update NULLs directly:

```sql
UPDATE orders
SET shipping_cost = 0.00
WHERE shipping_cost IS NULL;

UPDATE customers
SET country = 'US'
WHERE country IS NULL AND registration_region = 'north_america';
```

## Forward-Filling Missing Values

For time-series data, fill missing values using the previous known value with a subquery:

```sql
UPDATE readings r
JOIN (
  SELECT r1.id,
    (SELECT value FROM readings r2
     WHERE r2.sensor_id = r1.sensor_id
       AND r2.recorded_at < r1.recorded_at
       AND r2.value IS NOT NULL
     ORDER BY r2.recorded_at DESC
     LIMIT 1) AS prev_value
  FROM readings r1
  WHERE r1.value IS NULL
) filled ON r.id = filled.id
SET r.value = filled.prev_value;
```

## Setting NOT NULL Constraints

Once data is clean, enforce NOT NULL to prevent future missing values:

```sql
-- First, confirm no NULLs remain
SELECT COUNT(*) FROM customers WHERE email IS NULL;

-- Then add the constraint
ALTER TABLE customers MODIFY COLUMN email VARCHAR(255) NOT NULL DEFAULT '';
```

## Using DEFAULT Values

Define sensible defaults at the schema level:

```sql
CREATE TABLE orders (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  customer_id BIGINT NOT NULL,
  status VARCHAR(50) NOT NULL DEFAULT 'pending',
  notes TEXT DEFAULT NULL,
  priority TINYINT UNSIGNED NOT NULL DEFAULT 5
) ENGINE=InnoDB;
```

## Summary

Handle missing MySQL data by first auditing with `COUNT + CASE WHEN`, normalizing empty strings and placeholders to `NULL`, then filling with `COALESCE` for queries or targeted `UPDATE` statements for permanent fixes. Enforce `NOT NULL` constraints and schema-level `DEFAULT` values to prevent missing data from reappearing.
