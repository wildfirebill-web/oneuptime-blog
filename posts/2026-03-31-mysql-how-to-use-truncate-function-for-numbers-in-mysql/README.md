# How to Use TRUNCATE() Function for Numbers in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, TRUNCATE, Math Functions, Numeric Functions, SQL

Description: Learn how to use the TRUNCATE() function in MySQL to truncate decimal numbers to a specified number of decimal places without rounding.

---

## What Is the TRUNCATE() Function?

`TRUNCATE(X, D)` truncates the number X to D decimal places. Unlike `ROUND()`, it never rounds - it simply removes digits beyond D decimal places. When D is negative, it truncates digits to the left of the decimal point.

```sql
SELECT TRUNCATE(123.456, 2);   -- Returns 123.45 (not 123.46!)
SELECT ROUND(123.456, 2);      -- Returns 123.46 (rounded up)
```

## Basic Syntax

```sql
-- Truncate to 2 decimal places
SELECT TRUNCATE(9.9999, 2);    -- Returns 9.99

-- Truncate to 0 decimal places (integer part only)
SELECT TRUNCATE(9.9, 0);       -- Returns 9 (not 10!)

-- Truncate to negative decimal places (zeroes out digits)
SELECT TRUNCATE(123.456, -1);  -- Returns 120
SELECT TRUNCATE(123.456, -2);  -- Returns 100
SELECT TRUNCATE(123.456, -3);  -- Returns 0

-- With negative numbers
SELECT TRUNCATE(-9.99, 1);     -- Returns -9.9
```

## TRUNCATE vs ROUND

```sql
-- The key difference: TRUNCATE always removes digits, ROUND may round up
SELECT TRUNCATE(2.9999, 2);    -- Returns 2.99 (no rounding)
SELECT ROUND(2.9999, 2);       -- Returns 3.00 (rounds up)

SELECT TRUNCATE(-2.9999, 2);   -- Returns -2.99
SELECT ROUND(-2.9999, 2);      -- Returns -3.00

-- Financial calculations: TRUNCATE is safer for floor-based billing
SELECT TRUNCATE(19.999, 2);    -- Returns 19.99 (customer pays less)
SELECT ROUND(19.999, 2);       -- Returns 20.00 (customer pays more)
```

## Practical Financial Examples

```sql
-- Calculate tax without rounding up (floor-based)
SELECT
    product_name,
    price,
    TRUNCATE(price * 0.08, 2) AS tax_floor,
    ROUND(price * 0.08, 2) AS tax_rounded
FROM products;

-- Truncate to cents in billing
SELECT
    order_id,
    subtotal,
    TRUNCATE(subtotal * 1.08, 2) AS total_with_tax
FROM orders;

-- Calculate discount and truncate to whole dollars
SELECT
    customer_id,
    total_purchases,
    TRUNCATE(total_purchases * 0.05, 0) AS discount_dollars
FROM customers
WHERE total_purchases > 1000;
```

## Using TRUNCATE in Data Bucketing

```sql
-- Group prices into $10 buckets
SELECT
    TRUNCATE(price, -1) AS price_bucket,
    COUNT(*) AS product_count
FROM products
GROUP BY TRUNCATE(price, -1)
ORDER BY price_bucket;

-- Group timestamps into 15-minute intervals
SELECT
    TRUNCATE(UNIX_TIMESTAMP(event_time) / 900, 0) * 900 AS interval_start,
    COUNT(*) AS events
FROM events
GROUP BY interval_start
ORDER BY interval_start;
```

## TRUNCATE with Integer Division

```sql
-- Truncate to thousands (effectively integer division by 1000)
SELECT TRUNCATE(45678.90, -3);   -- Returns 45000

-- Useful for rounding metrics to significant figures
SELECT
    user_id,
    TRUNCATE(total_bytes / 1048576, 2) AS storage_mb
FROM storage_usage;
```

## TRUNCATE vs FLOOR

```sql
-- FLOOR always rounds toward negative infinity
-- TRUNCATE always removes digits (rounds toward zero)

SELECT FLOOR(4.9);      -- Returns 4
SELECT TRUNCATE(4.9, 0); -- Returns 4 (same for positive)

SELECT FLOOR(-4.1);     -- Returns -5 (toward -infinity)
SELECT TRUNCATE(-4.1, 0); -- Returns -4 (toward zero)

-- For positive numbers: TRUNCATE(x, 0) = FLOOR(x)
-- For negative numbers: they differ
```

## Using TRUNCATE in Stored Procedures

```sql
DELIMITER //
CREATE FUNCTION calculate_installment(
    principal DECIMAL(12, 2),
    rate DECIMAL(5, 4),
    months INT
) RETURNS DECIMAL(10, 2) DETERMINISTIC
BEGIN
    DECLARE monthly_rate DECIMAL(10, 8);
    DECLARE payment DECIMAL(12, 6);
    SET monthly_rate = rate / 12;
    SET payment = principal * (monthly_rate * POW(1 + monthly_rate, months))
                  / (POW(1 + monthly_rate, months) - 1);
    -- Truncate (not round) to 2 decimal places for conservative calculation
    RETURN TRUNCATE(payment, 2);
END //
DELIMITER ;
```

## Summary

`TRUNCATE(X, D)` removes decimal digits beyond D places without rounding, making it the right choice when you need "floor" behavior rather than standard rounding. Use it for financial calculations where rounding up is undesirable, for bucketing data into ranges, and for generating clean integers from decimal expressions. Remember that negative values of D truncate digits to the left of the decimal point, effectively zeroing out tens, hundreds, or thousands.
