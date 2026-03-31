# How to Use log(), log2(), log10() Functions in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Math Function, Logarithm, Data Analysis

Description: Learn how log(), log2(), and log10() compute logarithms in ClickHouse, with examples for entropy calculation, compression ratio, and logarithmic scale visualization.

---

ClickHouse provides three logarithm functions covering the most common bases. `log()` computes the natural logarithm (base e), `log2()` computes the binary logarithm (base 2), and `log10()` computes the common logarithm (base 10). Logarithms appear in information theory, data compression analysis, visualization scaling, statistical distributions, and feature engineering for machine learning. Understanding which base to use for a given problem is key to producing meaningful results.

## Function Signatures

```text
log(x)    -- natural logarithm, base e; also ln(x)
log2(x)   -- binary logarithm, base 2
log10(x)  -- common logarithm, base 10
```

All three return Float64. Inputs must be positive; log(0) and log of negative numbers return -Inf or NaN.

## Basic Usage

```sql
SELECT
    round(log(1),    4)  AS ln_1,
    round(log(exp(1)), 4) AS ln_e,
    round(log(100),  4)  AS ln_100,
    round(log2(8),   4)  AS log2_8,
    round(log2(1024),4)  AS log2_1024,
    round(log10(100),4)  AS log10_100,
    round(log10(1000),4) AS log10_1000;
```

`log(1)` = 0, `log(e)` = 1, `log2(8)` = 3, `log10(100)` = 2.

## Changing Logarithm Base

Any base can be computed using the change of base formula: log_b(x) = log(x) / log(b).

```sql
SELECT
    round(log(243) / log(3), 4) AS log_base3_243
;
```

## Shannon Entropy Calculation

Shannon entropy measures the information content of a distribution. For a probability p, the contribution is `-p * log2(p)`. Use `log2()` to compute entropy in bits.

```sql
CREATE TABLE class_distribution
(
    label       String,
    event_count UInt64
)
ENGINE = MergeTree
ORDER BY label;

INSERT INTO class_distribution VALUES
('click',    4500),
('view',     3200),
('purchase',  800),
('bounce',   1500);
```

```sql
WITH totals AS (
    SELECT sum(event_count) AS total FROM class_distribution
)
SELECT
    round(
        -sum((event_count / t.total) * log2(event_count / t.total)),
        4
    ) AS shannon_entropy_bits
FROM class_distribution, totals t;
```

Higher entropy means a more uniform distribution; lower entropy means the distribution is more concentrated in one class.

## Computing Log-Odds for Logistic Regression

Use `log()` to compute log-odds from probabilities, a key transformation in logistic regression and click-through-rate models.

```sql
CREATE TABLE ctr_data
(
    segment     String,
    impressions UInt64,
    clicks      UInt64
)
ENGINE = MergeTree
ORDER BY segment;

INSERT INTO ctr_data VALUES
('mobile',   50000, 1200),
('desktop',  80000, 1600),
('tablet',   15000,  450);
```

```sql
SELECT
    segment,
    round(clicks / impressions, 4)                        AS ctr,
    round(log(clicks / (impressions - clicks)), 4)        AS log_odds
FROM ctr_data;
```

## Compression Ratio Analysis

Use `log2()` to measure the effective bit depth of compressed data. The number of bits needed to represent N distinct values is `ceil(log2(N))`.

```sql
SELECT
    distinct_values,
    ceil(log2(distinct_values)) AS bits_needed
FROM (
    SELECT arrayJoin([2, 8, 256, 1000, 65536, 1048576]) AS distinct_values
);
```

## Logarithmic Scale for Visualization

Apply `log10()` to skewed distributions to improve visualization. Values spanning many orders of magnitude (like revenue or population) are often better displayed on a log scale.

```sql
CREATE TABLE company_revenue
(
    company_name String,
    revenue      Float64
)
ENGINE = MergeTree
ORDER BY company_name;

INSERT INTO company_revenue VALUES
('startup_a',     50000.0),
('startup_b',    250000.0),
('mid_corp_x',  5000000.0),
('large_corp_y', 500000000.0),
('enterprise_z', 25000000000.0);
```

```sql
SELECT
    company_name,
    revenue,
    round(log10(revenue), 2) AS log10_revenue,
    floor(log10(revenue))    AS order_of_magnitude
FROM company_revenue
ORDER BY revenue;
```

## Geometric Mean Using log()

The geometric mean is `exp(avg(log(x)))`, which is more numerically stable than computing the product then taking the nth root.

```sql
SELECT
    round(exp(avg(log(revenue))), 2) AS geometric_mean_revenue
FROM company_revenue;
```

## Summary

`log()`, `log2()`, and `log10()` cover the three most common logarithm bases in ClickHouse. Use `log()` (natural log) for growth rates, log-odds, and exponential distribution parameters. Use `log2()` for information entropy, bit-depth calculations, and binary tree depths. Use `log10()` for order-of-magnitude analysis and log-scale display. To compute a logarithm in any other base, apply the change-of-base formula: `log(x) / log(base)`. Combine with `exp()` for the geometric mean pattern and with `ceil()` for minimum bit-width calculations.
