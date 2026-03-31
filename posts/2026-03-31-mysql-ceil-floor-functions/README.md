# How to Use CEIL() and FLOOR() Functions in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, SQL, Math Function, Database

Description: Learn how MySQL's CEIL() and FLOOR() functions round a number up or down to the nearest integer, with practical examples for pagination and pricing.

---

## What are CEIL() and FLOOR()?

MySQL provides two rounding functions that round to the nearest integer in opposite directions:

- `CEIL(X)` (also `CEILING(X)`) - returns the smallest integer not less than `X` (rounds up)
- `FLOOR(X)` - returns the largest integer not greater than `X` (rounds down)

Both functions return an integer value. They are essential in pagination calculations, pricing logic, bucket assignment, and any scenario requiring directed rounding.

```sql
CEIL(X)
CEILING(X)  -- alias
FLOOR(X)
```

## Basic Examples

```sql
SELECT CEIL(4.1);    -- Result: 5
SELECT CEIL(4.9);    -- Result: 5
SELECT CEIL(-4.1);   -- Result: -4  (rounds toward zero)
SELECT CEIL(-4.9);   -- Result: -4

SELECT FLOOR(4.9);   -- Result: 4
SELECT FLOOR(4.1);   -- Result: 4
SELECT FLOOR(-4.1);  -- Result: -5  (rounds away from zero)
SELECT FLOOR(-4.9);  -- Result: -5
```

## Behavior with Negative Numbers

Negative number rounding differs from positive:

- `CEIL(-4.3)` = `-4` (rounds up toward zero)
- `FLOOR(-4.3)` = `-5` (rounds down away from zero)

## Pagination - Total Page Count

Calculate the number of pages needed to display all rows:

```sql
SELECT CEIL(COUNT(*) / 10.0) AS total_pages
FROM products;
```

This ensures the last page is counted even when rows don't divide evenly.

## Price Rounding - Always Round Up

For pricing where fractions are always billed to the next whole unit:

```sql
SELECT
  product_name,
  unit_price,
  quantity,
  CEIL(unit_price * quantity) AS total_charged
FROM order_items;
```

## Age Calculation - Whole Years Elapsed

```sql
SELECT
  name,
  FLOOR(DATEDIFF(CURDATE(), birthdate) / 365.25) AS age_years
FROM persons;
```

`FLOOR()` ensures we only count completed years.

## Bucketing Data into Ranges

Group values into equal-width buckets using `FLOOR()`:

```sql
SELECT
  FLOOR(score / 10) * 10 AS bucket_start,
  COUNT(*) AS count
FROM test_results
GROUP BY bucket_start
ORDER BY bucket_start;
```

This groups scores into 0-9, 10-19, 20-29, etc.

## Rounding Prices to Nearest Dollar

```sql
-- Round up all prices to next dollar
SELECT product_name, CEIL(price) AS price_rounded_up FROM products;

-- Round down all prices (floor price)
SELECT product_name, FLOOR(price) AS floor_price FROM products;
```

## NULL Handling

Both functions return `NULL` for `NULL` input:

```sql
SELECT CEIL(NULL);   -- Result: NULL
SELECT FLOOR(NULL);  -- Result: NULL
```

## CEIL() vs ROUND()

- `ROUND(4.5)` = 5 (rounds to nearest, ties round up)
- `CEIL(4.1)` = 5 (always rounds up regardless)
- `FLOOR(4.9)` = 4 (always rounds down regardless)

Use `CEIL()` when you must guarantee the result is at least as large as the input, and `FLOOR()` when you need the result to never exceed the input.

## Summary

`CEIL()` rounds up to the nearest integer and `FLOOR()` rounds down. For negative values, `CEIL()` rounds toward zero and `FLOOR()` rounds away from zero. Key use cases include pagination (total page count with `CEIL()`), age computation (completed years with `FLOOR()`), price ceilings, and equal-width data bucketing. Both return `NULL` for `NULL` input.
