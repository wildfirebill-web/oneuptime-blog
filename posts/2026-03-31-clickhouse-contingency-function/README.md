# How to Use contingency() Function in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, Aggregate Function, Contingency, Statistics

Description: Learn how to use the contingency() aggregate function in ClickHouse to measure the association between two categorical variables using the contingency coefficient.

---

The contingency coefficient is a classical measure of association between two categorical variables, computed from the chi-squared statistic. ClickHouse's `contingency()` aggregate function calculates this coefficient directly in SQL, providing a normalized value that indicates how strongly two categorical variables are related. It is symmetric, easy to interpret, and works on any string or numeric categorical columns.

## What Is the Contingency Coefficient

The contingency coefficient C is defined as:

```text
C = sqrt(chi2 / (chi2 + n))
```

Where `chi2` is the Pearson chi-squared statistic and `n` is the total number of observations.

Key properties:
- Always between 0 and 1
- **0** - variables are independent
- **Higher values** - stronger association
- The maximum value is less than 1 and depends on the table dimensions

## Syntax

```sql
contingency(x, y)
```

Accepts two categorical columns and returns a `Float64` in [0, 1).

## Creating Sample Data

```sql
CREATE TABLE customer_data
(
    customer_id  UInt32,
    region       String,
    product_tier String
)
ENGINE = MergeTree()
ORDER BY customer_id;

INSERT INTO customer_data VALUES
    (1,  'north', 'basic'),
    (2,  'north', 'basic'),
    (3,  'north', 'premium'),
    (4,  'south', 'premium'),
    (5,  'south', 'enterprise'),
    (6,  'south', 'enterprise'),
    (7,  'east',  'basic'),
    (8,  'east',  'premium'),
    (9,  'west',  'enterprise'),
    (10, 'north', 'basic'),
    (11, 'south', 'premium'),
    (12, 'east',  'basic'),
    (13, 'west',  'premium'),
    (14, 'north', 'enterprise'),
    (15, 'west',  'enterprise');
```

## Basic Usage

```sql
SELECT contingency(region, product_tier)
FROM customer_data;
```

## Interpreting the Result

```sql
SELECT
    contingency(region, product_tier)  AS c,
    multiIf(
        c < 0.1, 'negligible',
        c < 0.3, 'weak',
        c < 0.5, 'moderate',
        'strong'
    )                                  AS interpretation
FROM customer_data;
```

Note that the maximum possible value of the contingency coefficient is less than 1 for any finite contingency table. For a 2x2 table, the maximum is approximately 0.707.

## Comparing Multiple Variable Pairs

```sql
SELECT
    contingency(region, product_tier)      AS region_vs_tier,
    contingency(region, channel)           AS region_vs_channel,
    contingency(product_tier, channel)     AS tier_vs_channel
FROM customer_data_extended;
```

## Contingency vs. Cramer's V

```sql
SELECT
    contingency(region, product_tier)              AS contingency_coeff,
    cramersV(region, product_tier)                 AS cramers_v,
    cramersVBiasCorrected(region, product_tier)    AS cramers_v_corrected
FROM customer_data;
```

Both the contingency coefficient and Cramer's V are derived from chi-squared. The key difference is that Cramer's V normalizes by the minimum dimension of the contingency table, making it comparable across tables of different sizes. The contingency coefficient does not have this normalization, so its maximum value varies by table dimensions.

## Churn Analysis: Which Features Associate with Churn

```sql
SELECT
    'region'       AS feature,
    contingency(region, churned) AS association
FROM customer_churn

UNION ALL

SELECT
    'product_tier' AS feature,
    contingency(product_tier, churned) AS association
FROM customer_churn

UNION ALL

SELECT
    'signup_channel' AS feature,
    contingency(signup_channel, churned) AS association
FROM customer_churn

ORDER BY association DESC;
```

## Segmented Analysis

```sql
SELECT
    month,
    contingency(device_type, plan_tier) AS device_plan_association
FROM user_behavior
GROUP BY month
ORDER BY month;
```

Tracking the contingency coefficient over time reveals whether the relationship between two categorical variables is strengthening or weakening.

## Use Case: Validating Data Independence

Use `contingency()` to verify that two categorical features are independent before including them both in a model:

```sql
SELECT
    contingency(feature_a, feature_b) AS c,
    if(c < 0.1, 'independent - safe to use both',
                'correlated - consider removing one') AS recommendation
FROM training_dataset;
```

## Summary

The `contingency()` function in ClickHouse computes the contingency coefficient, a symmetric measure of association between two categorical variables based on the chi-squared statistic. It returns a value between 0 and 1, where 0 indicates independence and higher values indicate stronger association. Use it alongside `cramersV()` for categorical association analysis, feature selection, and data quality validation in SQL without any external tooling.
