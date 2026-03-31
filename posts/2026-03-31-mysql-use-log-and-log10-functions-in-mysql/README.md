# How to Use LOG() and LOG10() Functions in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Log Function, Log10, Math Functions, Sql

Description: Learn how to use LOG() and LOG10() mathematical functions in MySQL for logarithmic calculations, data normalization, and exponential growth analysis.

---

## Introduction

MySQL provides several logarithm functions: `LOG()` for natural logarithm or logarithm with a custom base, `LOG10()` for base-10 logarithm, and `LOG2()` for base-2 logarithm. These functions are useful for data normalization, scientific calculations, measuring exponential growth, and converting multiplicative relationships to additive ones.

## LOG() - Natural Logarithm

```sql
-- Natural logarithm (base e)
SELECT LOG(number);

-- Logarithm with custom base
SELECT LOG(base, number);
```

## LOG10() - Base-10 Logarithm

```sql
SELECT LOG10(number);
```

## LOG2() - Base-2 Logarithm

```sql
SELECT LOG2(number);
```

## Basic Examples

```sql
SELECT LOG(1);      -- Returns: 0     (ln(1) = 0)
SELECT LOG(2.718);  -- Returns: ~1    (ln(e) ≈ 1)
SELECT LOG(10);     -- Returns: ~2.30 (ln(10))

SELECT LOG10(1);    -- Returns: 0     (log10(1) = 0)
SELECT LOG10(10);   -- Returns: 1     (log10(10) = 1)
SELECT LOG10(100);  -- Returns: 2
SELECT LOG10(1000); -- Returns: 3

SELECT LOG2(1);     -- Returns: 0
SELECT LOG2(2);     -- Returns: 1
SELECT LOG2(8);     -- Returns: 3
SELECT LOG2(1024);  -- Returns: 10
```

## LOG with Custom Base

```sql
-- log base 5 of 25
SELECT LOG(5, 25);   -- Returns: 2   (5^2 = 25)

-- log base 2 of 16
SELECT LOG(2, 16);   -- Returns: 4   (2^4 = 16)

-- Equivalent to LOG2
SELECT LOG(2, 1024); -- Returns: 10
```

## Logarithm and NULL/Zero

```sql
SELECT LOG(0);      -- Returns: NULL (undefined: log(0) is -infinity)
SELECT LOG(-5);     -- Returns: NULL (undefined: log of negative)
SELECT LOG10(0);    -- Returns: NULL
SELECT LOG(1, 10);  -- Returns: NULL (undefined: log base 1)
```

## Data Normalization with LOG

Log normalization reduces the effect of large outliers in data:

```sql
SELECT
  product_name,
  page_views,
  LOG10(page_views + 1) AS log_views
FROM product_stats
ORDER BY log_views DESC;
```

Adding 1 before logging avoids `LOG(0)` issues.

## Richter Scale Calculation

Logarithms are used in many scientific scales:

```sql
-- Calculate Richter scale difference between two earthquakes
SELECT LOG10(amplitude_large / amplitude_small) AS richter_difference
FROM earthquake_data;
```

## Population Growth Analysis

```sql
-- Calculate annual growth rate using log
-- If population grew from start_pop to end_pop over years
SELECT
  city,
  start_population,
  end_population,
  years,
  (LOG(end_population) - LOG(start_population)) / years AS annual_growth_rate
FROM population_data;
```

## Reverse: Using EXP() and POW()

`LOG` and `EXP` are inverse functions:

```sql
SELECT EXP(LOG(5));    -- Returns: 5
SELECT LOG(EXP(3));    -- Returns: 3
SELECT POW(10, LOG10(100)); -- Returns: 100
```

## Logarithm Base Conversion

Convert between bases using the change-of-base formula: `log_b(x) = ln(x) / ln(b)`.

```sql
-- LOG base 7 of 49 using natural log
SELECT LOG(49) / LOG(7);  -- Returns: 2

-- Verify with custom base form
SELECT LOG(7, 49);        -- Returns: 2
```

## Order of Magnitude Classification

```sql
SELECT
  id,
  value,
  FLOOR(LOG10(value)) AS order_of_magnitude
FROM measurements
WHERE value > 0;
-- value=5 -> 0, value=50 -> 1, value=500 -> 2, value=5000 -> 3
```

## Practical Example: Scoring Algorithm

Log-based scoring to dampen large differences:

```sql
SELECT
  article_id,
  view_count,
  upvotes,
  LOG(view_count + 1) + LOG(upvotes + 1) AS popularity_score
FROM articles
ORDER BY popularity_score DESC
LIMIT 10;
```

## Summary

`LOG()` computes the natural logarithm or a custom base logarithm. `LOG10()` computes the base-10 logarithm. `LOG2()` computes the base-2 logarithm. These functions are useful for data normalization, exponential growth analysis, scientific calculations, and dampening the effect of outliers. Remember that logarithms of zero and negative numbers return NULL.
