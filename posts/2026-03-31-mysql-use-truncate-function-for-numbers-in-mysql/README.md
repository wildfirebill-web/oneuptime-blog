# How to Use TRUNCATE() Function for Numbers in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Truncate Function, Math Functions, Sql, Numeric

Description: Learn how to use the TRUNCATE() function in MySQL to truncate a number to a specified number of decimal places without rounding, with practical examples.

---

## Introduction

`TRUNCATE()` is a MySQL mathematical function that removes digits from a number beyond a specified decimal position, without rounding. Unlike `ROUND()`, which rounds to the nearest value, `TRUNCATE()` simply discards the unwanted digits. This distinction is critical in financial, statistical, and measurement applications.

## Basic Syntax

```sql
TRUNCATE(number, decimal_places)
```

- `number`: the numeric value to truncate.
- `decimal_places`: how many decimal places to keep. Can be negative to truncate to the left of the decimal point.

## Basic Examples

```sql
SELECT TRUNCATE(7.8956, 2);   -- Returns: 7.89  (not 7.90)
SELECT TRUNCATE(7.8956, 1);   -- Returns: 7.8
SELECT TRUNCATE(7.8956, 0);   -- Returns: 7
SELECT TRUNCATE(7.8956, -1);  -- Returns: 0      (truncates to tens)
SELECT TRUNCATE(128.9, -2);   -- Returns: 100    (truncates to hundreds)
```

## TRUNCATE vs ROUND

This is the most important distinction:

```sql
SELECT ROUND(7.895, 2);     -- Returns: 7.90 (rounds up)
SELECT TRUNCATE(7.895, 2);  -- Returns: 7.89 (truncates, no rounding)

SELECT ROUND(7.845, 2);     -- Returns: 7.85 (rounds to nearest even or up)
SELECT TRUNCATE(7.845, 2);  -- Returns: 7.84 (always truncates toward zero)
```

Use `TRUNCATE` when you need to drop precision without any rounding effect.

## Negative Numbers

```sql
SELECT TRUNCATE(-7.8956, 2);  -- Returns: -7.89  (truncates toward zero)
SELECT TRUNCATE(-7.8956, 0);  -- Returns: -7
```

`TRUNCATE` always truncates toward zero, not toward negative infinity.

## Financial Calculations

Truncate currency values to cents (2 decimal places):

```sql
SELECT
  product_name,
  price,
  TRUNCATE(price * 0.9, 2) AS discounted_price
FROM products;
```

Truncate vs round makes a difference when processing many transactions - truncating always favors the correct floor value.

## Truncating to the Left of Decimal

Negative decimal places truncate to powers of 10:

```sql
SELECT TRUNCATE(12345, -1);  -- Returns: 12340
SELECT TRUNCATE(12345, -2);  -- Returns: 12300
SELECT TRUNCATE(12345, -3);  -- Returns: 12000
SELECT TRUNCATE(12345, -4);  -- Returns: 10000
SELECT TRUNCATE(12345, -5);  -- Returns: 0
```

Useful for rounding to the nearest 10, 100, 1000 while always going down.

## Calculating Age in Full Years

```sql
SELECT
  name,
  TRUNCATE(DATEDIFF(NOW(), birth_date) / 365.25, 0) AS age_years
FROM users;
```

This gives the floor age (completed years), not a rounded age.

## Floor vs TRUNCATE

For positive numbers, `TRUNCATE(x, 0)` and `FLOOR(x)` behave the same. For negative numbers they differ:

```sql
SELECT FLOOR(-7.3);         -- Returns: -8 (floor rounds toward negative infinity)
SELECT TRUNCATE(-7.3, 0);   -- Returns: -7 (truncate rounds toward zero)
```

## Percentage Calculations

```sql
SELECT
  product_name,
  sold_count,
  total_count,
  TRUNCATE(sold_count / total_count * 100, 1) AS sold_pct
FROM inventory_stats;
```

## Using TRUNCATE in ORDER BY

```sql
SELECT product_name, price, TRUNCATE(price, 0) AS price_floor
FROM products
ORDER BY price_floor ASC;
```

## TRUNCATE with NULL

```sql
SELECT TRUNCATE(NULL, 2);  -- Returns: NULL
SELECT TRUNCATE(5.99, NULL); -- Returns: NULL
```

## Batch Update: Normalize Prices

```sql
UPDATE products
SET price = TRUNCATE(price, 2)
WHERE price != TRUNCATE(price, 2);
```

This removes any extra decimal precision beyond 2 places.

## Summary

`TRUNCATE(number, decimal_places)` removes digits beyond the specified position without rounding, always truncating toward zero. Use it instead of `ROUND()` when you need to drop precision deterministically, such as in financial calculations, floor age computations, and data normalization. Negative decimal places let you truncate to tens, hundreds, or thousands.
