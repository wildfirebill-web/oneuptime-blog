# How to Use GREATEST and LEAST for Conditional Logic in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Conditional Function, Greatest, Least, Clamping

Description: Learn how to use GREATEST and LEAST in ClickHouse for row-level min/max comparisons, clamping values between bounds, and NULL-aware column selection.

---

`greatest(a, b, ...)` returns the largest value among its arguments. `least(a, b, ...)` returns the smallest. Unlike the aggregate functions `max()` and `min()` which operate across rows, `greatest()` and `least()` operate across columns within a single row. NULL arguments are ignored - if all arguments are NULL, the result is NULL. This makes them safe to use on nullable columns.

## Basic Usage

```sql
-- Greatest returns the largest value
SELECT greatest(3, 7, 5, 1) AS max_value;   -- returns 7

-- Least returns the smallest value
SELECT least(3, 7, 5, 1) AS min_value;      -- returns 1

-- Apply across columns in a row
SELECT
    order_id,
    price_a,
    price_b,
    price_c,
    greatest(price_a, price_b, price_c) AS max_price,
    least(price_a, price_b, price_c)    AS min_price
FROM price_quotes
LIMIT 10;
```

## Clamping Values Between Bounds

A very common use case is clamping a value so that it stays within a defined range. Use `greatest(lower_bound, least(upper_bound, value))`.

```sql
-- Clamp discount rate between 0% and 50%
SELECT
    product_id,
    raw_discount,
    greatest(0.0, least(0.5, raw_discount)) AS clamped_discount
FROM discount_inputs
LIMIT 10;

-- Clamp score between 0 and 100
SELECT
    user_id,
    raw_score,
    greatest(0, least(100, raw_score)) AS clamped_score
FROM test_results
LIMIT 10;
```

## Picking the Larger/Smaller of Two Columns

`greatest` and `least` with two arguments work like a row-level conditional max/min.

```sql
-- Use the higher of two bids as the effective price
SELECT
    auction_id,
    bid_a,
    bid_b,
    greatest(bid_a, bid_b) AS winning_bid
FROM auction_bids
LIMIT 10;

-- Prefer the earlier of two timestamps
SELECT
    event_id,
    client_timestamp,
    server_timestamp,
    least(client_timestamp, server_timestamp) AS earliest_timestamp
FROM events
LIMIT 10;
```

## NULL Handling

`greatest` and `least` skip NULL values when comparing. If only one non-NULL value exists, it is returned. If all values are NULL, the result is NULL.

```sql
-- NULL is ignored: greatest(1, NULL, 3) = 3
SELECT greatest(1, NULL, 3) AS result;   -- returns 3
SELECT least(NULL, NULL)    AS result;   -- returns NULL

-- Practical example: pick the best available price, ignoring missing prices
SELECT
    product_id,
    least(
        ifNull(supplier_a_price, 999999),
        ifNull(supplier_b_price, 999999),
        ifNull(supplier_c_price, 999999)
    ) AS best_price
FROM supplier_prices
LIMIT 10;
```

## Range Validation

Use `greatest` and `least` to validate that values fall within expected ranges.

```sql
-- Flag rows where the observed value is outside the expected range
SELECT
    sensor_id,
    reading,
    expected_min,
    expected_max,
    reading < expected_min OR reading > expected_max AS out_of_range,
    -- Compute how far out of range the value is
    greatest(0, reading - expected_max)               AS above_max_by,
    greatest(0, expected_min - reading)               AS below_min_by
FROM sensor_readings
WHERE out_of_range = 1
LIMIT 20;
```

## Computing Row-Level Ranges

```sql
-- Compute the range (max - min) across multiple measurement columns
SELECT
    experiment_id,
    measurement_1,
    measurement_2,
    measurement_3,
    greatest(measurement_1, measurement_2, measurement_3) -
    least(measurement_1, measurement_2, measurement_3) AS measurement_range
FROM experiments
LIMIT 10;
```

## Using greatest and least in WHERE Clauses

```sql
-- Find rows where at least one measurement exceeds the threshold
SELECT *
FROM sensor_data
WHERE greatest(temp_a, temp_b, temp_c) > 100
LIMIT 10;

-- Find rows where all measurements are within bounds
SELECT *
FROM sensor_data
WHERE least(temp_a, temp_b, temp_c) >= 0
  AND greatest(temp_a, temp_b, temp_c) <= 100
LIMIT 10;
```

## Summary

`greatest()` and `least()` provide row-level comparison across multiple columns or expressions. They are NULL-safe (NULLs are ignored in the comparison), unlike the aggregate `max()` and `min()` which also ignore NULLs but operate vertically across rows. The most common patterns are clamping values between bounds with `greatest(lower, least(upper, value))`, and picking the best available non-null value by substituting a sentinel large or small number for NULLs before comparing.
