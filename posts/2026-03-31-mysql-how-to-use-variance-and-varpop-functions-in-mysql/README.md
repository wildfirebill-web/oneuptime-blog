# How to Use VARIANCE() and VAR_POP() Functions in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Variance, Var Pop, Statistical Functions, Aggregate Functions

Description: Learn how to use MySQL's VARIANCE() and VAR_POP() functions to calculate statistical variance of a dataset, including the difference between population and sample variance.

---

## Overview

MySQL provides two aggregate functions for calculating statistical variance: `VAR_POP()` (also called `VARIANCE()`) computes the population variance, and `VAR_SAMP()` computes the sample variance. Variance measures how spread out values are from the mean. These functions are essential for statistical analysis, quality control, and anomaly detection directly in SQL.

## Function Reference

```sql
VARIANCE(expr)   -- alias for VAR_POP() - population variance
VAR_POP(expr)    -- population variance (divides by N)
VAR_SAMP(expr)   -- sample variance (divides by N-1)
STDDEV_POP(expr) -- population standard deviation (sqrt of VAR_POP)
STDDEV(expr)     -- alias for STDDEV_POP
STDDEV_SAMP(expr)-- sample standard deviation (sqrt of VAR_SAMP)
```

## Population vs Sample Variance

```sql
-- Population variance: use when you have the ENTIRE population
-- Formula: SUM((x - mean)^2) / N
SELECT VAR_POP(score) AS population_variance FROM scores;

-- Sample variance: use when you have a SAMPLE of a larger population
-- Formula: SUM((x - mean)^2) / (N-1)
SELECT VAR_SAMP(score) AS sample_variance FROM scores;
```

Use `VAR_POP()` when your data represents the complete set (e.g., all sales last month). Use `VAR_SAMP()` when your data is a sample drawn from a larger population.

## Basic Examples

```sql
CREATE TABLE test_scores (
  student_id INT,
  subject VARCHAR(50),
  score DECIMAL(5,2)
);

INSERT INTO test_scores VALUES
(1, 'Math',    85),
(2, 'Math',    90),
(3, 'Math',    78),
(4, 'Math',    92),
(5, 'Math',    88),
(1, 'Science', 75),
(2, 'Science', 80),
(3, 'Science', 95),
(4, 'Science', 70),
(5, 'Science', 85);

-- Basic statistics per subject
SELECT
  subject,
  COUNT(*)          AS n,
  AVG(score)        AS mean,
  VAR_POP(score)    AS pop_variance,
  VAR_SAMP(score)   AS sample_variance,
  STDDEV_POP(score) AS pop_stddev,
  STDDEV_SAMP(score)AS sample_stddev,
  MIN(score)        AS min_score,
  MAX(score)        AS max_score
FROM test_scores
GROUP BY subject;
```

## Detecting High-Variance Groups

```sql
-- Find products with high price variance across stores
CREATE TABLE store_prices (
  product_id INT,
  store_id INT,
  price DECIMAL(10,2)
);

INSERT INTO store_prices VALUES
(1, 1, 10.00), (1, 2, 10.50), (1, 3, 10.25),
(2, 1, 5.00),  (2, 2, 8.00),  (2, 3, 12.00);  -- high variance

SELECT
  product_id,
  ROUND(AVG(price), 2) AS avg_price,
  ROUND(VAR_POP(price), 4) AS price_variance,
  ROUND(STDDEV_POP(price), 2) AS price_stddev
FROM store_prices
GROUP BY product_id
HAVING VAR_POP(price) > 2
ORDER BY price_variance DESC;
```

## Quality Control: Detecting Measurement Inconsistency

```sql
-- Manufacturing measurements - flag batches with high variance
CREATE TABLE measurements (
  batch_id INT,
  dimension DECIMAL(8,4)
);

INSERT INTO measurements VALUES
(1, 10.001), (1, 10.002), (1, 9.999), (1, 10.001),
(2, 10.001), (2, 10.500), (2, 9.500), (2, 10.002);  -- defective batch

SELECT
  batch_id,
  ROUND(AVG(dimension), 4)     AS mean_dim,
  ROUND(STDDEV_POP(dimension), 6) AS stddev,
  CASE WHEN STDDEV_POP(dimension) > 0.1 THEN 'FAIL' ELSE 'PASS' END AS qc_result
FROM measurements
GROUP BY batch_id;
```

## Time-Series Variance Analysis

```sql
-- Calculate weekly variance of daily sales
SELECT
  YEAR(sale_date) AS yr,
  WEEK(sale_date) AS wk,
  ROUND(AVG(amount), 2)        AS avg_daily,
  ROUND(STDDEV_POP(amount), 2) AS stddev_daily,
  ROUND(VAR_POP(amount), 2)    AS variance
FROM daily_sales
GROUP BY YEAR(sale_date), WEEK(sale_date)
ORDER BY yr, wk;
```

## Coefficient of Variation (Relative Variability)

```sql
-- Coefficient of variation (CV) = stddev / mean * 100
-- Useful for comparing variability across groups with different scales
SELECT
  subject,
  ROUND(AVG(score), 2) AS mean,
  ROUND(STDDEV_POP(score), 2) AS stddev,
  ROUND(STDDEV_POP(score) / AVG(score) * 100, 2) AS cv_percent
FROM test_scores
GROUP BY subject
ORDER BY cv_percent DESC;
```

## NULL Handling

```sql
-- Both functions ignore NULL values
SELECT VAR_POP(score) FROM test_scores WHERE score IS NULL;  -- NULL (no rows)
SELECT VAR_POP(score) FROM (SELECT NULL AS score) t;         -- NULL
SELECT VAR_POP(score) FROM (SELECT 5 AS score) t;            -- 0 (single value)
```

## Summary

`VARIANCE()` and `VAR_POP()` compute population variance (divide by N), while `VAR_SAMP()` computes sample variance (divide by N-1). Use population variance when analyzing a complete dataset and sample variance when analyzing a subset. These functions are invaluable for statistical reporting, quality control, anomaly detection, and understanding data spread within SQL queries without requiring external tools.
