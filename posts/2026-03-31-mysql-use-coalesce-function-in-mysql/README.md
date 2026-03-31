# How to Use COALESCE() Function in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Coalesce, Null Handling, Sql, Functions

Description: Learn how to use the COALESCE() function in MySQL to return the first non-NULL value from a list of expressions, simplifying NULL handling in queries.

---

## Introduction

`COALESCE()` is a standard SQL function supported by MySQL that returns the first non-NULL value from a list of arguments. Unlike `IFNULL()`, which only takes two arguments, `COALESCE()` can take any number of arguments, making it more flexible for handling multiple potential NULL sources.

## Syntax

```sql
COALESCE(value1, value2, ..., valueN)
```

Returns the first non-NULL value in the list. If all values are NULL, it returns NULL.

## Basic Examples

```sql
SELECT COALESCE(NULL, NULL, 'third', 'fourth');
-- Returns: 'third'

SELECT COALESCE(NULL, 42, NULL);
-- Returns: 42

SELECT COALESCE(NULL, NULL, NULL);
-- Returns: NULL

SELECT COALESCE('first', 'second');
-- Returns: 'first'
```

## Replacing NULL with a Default Value

```sql
SELECT
  customer_id,
  COALESCE(middle_name, '') AS middle_name,
  COALESCE(phone, 'N/A') AS phone
FROM customers;
```

## Multiple Fallback Values

A key advantage of `COALESCE` is chaining multiple fallback options.

```sql
SELECT
  customer_id,
  COALESCE(mobile_phone, home_phone, work_phone, 'No contact') AS best_contact
FROM customers;
```

MySQL tries each argument in order and returns the first non-NULL one.

## COALESCE in Arithmetic

NULL in arithmetic expressions returns NULL. Use `COALESCE` to provide safe defaults.

```sql
SELECT
  product_name,
  price * COALESCE(quantity, 0) AS total_value
FROM inventory;
```

## COALESCE in WHERE Clause

```sql
SELECT name, region
FROM customers
WHERE COALESCE(region, 'Unknown') = 'Unknown';
```

Or more index-friendly:

```sql
SELECT name, region
FROM customers
WHERE region IS NULL OR region = 'Unknown';
```

## COALESCE with Aggregate Functions

```sql
SELECT
  department,
  COALESCE(SUM(bonus), 0) AS total_bonus
FROM employees
GROUP BY department;
```

## COALESCE vs IFNULL

Both functions serve similar purposes for two arguments, but `COALESCE` is the ANSI SQL standard.

```sql
-- These produce the same result
SELECT COALESCE(score, 0) FROM results;
SELECT IFNULL(score, 0) FROM results;

-- Only COALESCE supports multiple arguments
SELECT COALESCE(col1, col2, col3, 'default') FROM t;
```

## COALESCE vs NULLIF

`NULLIF(a, b)` is the inverse - it returns NULL if `a` equals `b`. These can be combined:

```sql
-- Replace empty strings with NULL, then replace NULL with 'N/A'
SELECT COALESCE(NULLIF(phone, ''), 'N/A') AS phone
FROM contacts;
```

## COALESCE in UPDATE

```sql
UPDATE products
SET final_price = COALESCE(sale_price, discounted_price, regular_price)
WHERE final_price IS NULL;
```

## COALESCE with CASE

For complex logic, `COALESCE` is cleaner than `CASE` when picking from multiple optional columns.

```sql
-- With COALESCE
SELECT COALESCE(preferred_name, first_name) AS display_name FROM users;

-- Equivalent with CASE
SELECT CASE WHEN preferred_name IS NOT NULL THEN preferred_name ELSE first_name END AS display_name
FROM users;
```

## Type Compatibility

All arguments to `COALESCE` should be compatible types. MySQL will attempt implicit conversion, but mismatched types can cause unexpected results.

```sql
-- All args are strings - fine
SELECT COALESCE(NULL, 'text', 42);
-- Returns: 'text' (42 is implicitly cast to string context)
```

## Summary

`COALESCE()` is a powerful and standard way to handle NULL values in MySQL. It returns the first non-NULL value from any number of arguments, making it ideal for providing default values, chaining fallback columns, and making arithmetic NULL-safe. Prefer `COALESCE` over `IFNULL` when you need more than two options.
