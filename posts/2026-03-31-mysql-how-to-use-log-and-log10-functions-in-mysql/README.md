# How to Use LOG() and LOG10() Functions in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, LOG, LOG10, Math Functions, Logarithm, SQL

Description: Learn how to use the LOG() and LOG10() functions in MySQL to compute natural logarithms and base-10 logarithms for data analysis and mathematical calculations.

---

## Overview of Logarithm Functions in MySQL

MySQL provides three main logarithm functions:

- `LOG(X)` - natural logarithm (base e, approximately 2.71828) of X
- `LOG(B, X)` - logarithm of X to the base B
- `LOG10(X)` - logarithm of X to base 10
- `LOG2(X)` - logarithm of X to base 2

```sql
SELECT LOG(1);        -- Returns 0 (ln(1) = 0)
SELECT LOG(2.71828);  -- Returns ~1 (ln(e) = 1)
SELECT LOG10(100);    -- Returns 2 (10^2 = 100)
SELECT LOG10(1000);   -- Returns 3 (10^3 = 1000)
SELECT LOG2(8);       -- Returns 3 (2^3 = 8)
```

## LOG() - Natural Logarithm

```sql
-- Natural log examples
SELECT LOG(1);       -- 0
SELECT LOG(10);      -- 2.302585092994046
SELECT LOG(100);     -- 4.605170185988092
SELECT LOG(EXP(5));  -- 5 (inverse of EXP)

-- LOG returns NULL for non-positive inputs
SELECT LOG(0);    -- NULL
SELECT LOG(-5);   -- NULL

-- LOG with two arguments: LOG(base, value)
SELECT LOG(10, 100);   -- Returns 2 (log base 10 of 100)
SELECT LOG(2, 8);      -- Returns 3 (log base 2 of 8)
SELECT LOG(3, 27);     -- Returns 3 (log base 3 of 27)
```

## LOG10() - Base-10 Logarithm

```sql
SELECT LOG10(1);       -- 0
SELECT LOG10(10);      -- 1
SELECT LOG10(100);     -- 2
SELECT LOG10(1000);    -- 3
SELECT LOG10(0.1);     -- -1
SELECT LOG10(0.01);    -- -2

-- Equivalent to LOG(10, X)
SELECT LOG10(50);       -- 1.6989700043360188
SELECT LOG(10, 50);     -- 1.6989700043360188 (same)
```

## Practical Use Cases

### Calculating Order of Magnitude

```sql
-- Categorize values by order of magnitude
SELECT
    product_name,
    price,
    FLOOR(LOG10(price)) AS magnitude,
    CASE FLOOR(LOG10(price))
        WHEN 0 THEN 'Single digits ($1-$9)'
        WHEN 1 THEN 'Tens ($10-$99)'
        WHEN 2 THEN 'Hundreds ($100-$999)'
        WHEN 3 THEN 'Thousands ($1000-$9999)'
        ELSE 'Over $10000'
    END AS price_category
FROM products
WHERE price > 0;
```

### Logarithmic Scaling for Data Visualization

```sql
-- Normalize very large value ranges with log scaling
SELECT
    id,
    view_count,
    ROUND(LOG10(view_count + 1), 2) AS log_views,  -- +1 to handle 0
    ROUND(LOG(view_count + 1), 2) AS ln_views
FROM articles
WHERE view_count >= 0
ORDER BY log_views DESC;
```

### Calculating Compound Growth Rate

```sql
-- Calculate CAGR (Compound Annual Growth Rate) using logarithms
-- CAGR = EXP(LOG(final/initial) / years) - 1
SELECT
    company,
    initial_value,
    final_value,
    years,
    ROUND(
        (EXP(LOG(final_value / initial_value) / years) - 1) * 100,
        2
    ) AS cagr_pct
FROM company_financials
WHERE initial_value > 0 AND final_value > 0;
```

### Richter Scale and dB Calculations

```sql
-- Calculate decibels from power ratio
-- dB = 10 * LOG10(power2 / power1)
SELECT
    measurement_id,
    signal_power,
    reference_power,
    ROUND(10 * LOG10(signal_power / reference_power), 2) AS decibels
FROM signal_measurements
WHERE reference_power > 0 AND signal_power > 0;
```

### Entropy Calculation

```sql
-- Shannon entropy for probability distributions
SELECT
    category,
    probability,
    -1 * probability * LOG(probability) / LOG(2) AS entropy_contribution
FROM category_probabilities
WHERE probability > 0;
```

## Combining LOG with Other Functions

```sql
-- Convert natural log to log base 10 manually
-- LOG10(x) = LOG(x) / LOG(10)
SELECT LOG(100) / LOG(10);  -- Returns 2 (same as LOG10(100))

-- Convert between bases using change of base formula
-- LOG_b(x) = LOG(x) / LOG(b)
SELECT LOG(1000) / LOG(7);  -- LOG base 7 of 1000

-- Exponential decay calculation
SELECT
    time_elapsed_hours,
    initial_amount,
    ROUND(initial_amount * EXP(-0.1 * time_elapsed_hours), 4) AS remaining
FROM decay_data;
```

## Handling Edge Cases

```sql
-- Protect against NULL and invalid inputs
SELECT COALESCE(LOG10(NULLIF(value, 0)), 0) AS safe_log
FROM measurements;

-- Handle negative or zero values
SELECT
    id,
    value,
    IF(value > 0, LOG10(value), NULL) AS log_value
FROM data_points;
```

## Summary

`LOG()` and `LOG10()` are MySQL's logarithm functions for natural and base-10 logarithms respectively. They return NULL for zero or negative arguments. Use them for data normalization and visualization (log scaling), order of magnitude classification, financial growth rate calculations, and any domain that uses logarithmic scales (decibels, pH, Richter scale). Use `LOG(B, X)` for arbitrary base logarithms via the change-of-base approach.
