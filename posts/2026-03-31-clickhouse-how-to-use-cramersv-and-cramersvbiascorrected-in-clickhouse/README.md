# How to Use cramersV() and cramersVBiasCorrected() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Statistics, Correlation, Data Analysis, Cramer's V

Description: Learn how to use cramersV() and cramersVBiasCorrected() in ClickHouse to measure statistical association between categorical variables.

---

## Overview

`cramersV()` computes Cramer's V statistic, a measure of association between two categorical (nominal) variables. Values range from 0 (no association) to 1 (perfect association). ClickHouse also provides `cramersVBiasCorrected()` which applies a bias correction for small sample sizes.

## Basic Syntax

```sql
SELECT cramersV(col1, col2)
FROM table_name;
```

Both columns should be categorical - strings or low-cardinality integers work well.

## Interpreting the Result

| Value Range | Interpretation |
|-------------|---------------|
| 0.0 - 0.1   | Negligible association |
| 0.1 - 0.3   | Weak association |
| 0.3 - 0.5   | Moderate association |
| 0.5 - 1.0   | Strong association |

## Practical Example: Feature vs Outcome

Check whether a user's country is associated with their subscription plan:

```sql
SELECT cramersV(country, subscription_plan) AS association_strength
FROM users
WHERE signup_date >= '2025-01-01';
```

## Comparing cramersV() and cramersVBiasCorrected()

`cramersVBiasCorrected()` reduces overestimation of association strength for small samples:

```sql
SELECT
    cramersV(device_type, converted)              AS v_raw,
    cramersVBiasCorrected(device_type, converted) AS v_corrected
FROM conversion_events;
```

For large datasets the two values converge. For samples under a few thousand rows, the corrected version is more reliable.

## Screening Multiple Feature Pairs

```sql
SELECT
    'browser vs plan'    AS pair, cramersV(browser, plan) AS v FROM users
UNION ALL
SELECT
    'country vs plan'    AS pair, cramersV(country, plan) AS v FROM users
UNION ALL
SELECT
    'device vs plan'     AS pair, cramersV(device_type, plan) AS v FROM users
ORDER BY v DESC;
```

## Table Setup for Example

```sql
CREATE TABLE users (
    user_id           UInt64,
    country           LowCardinality(String),
    browser           LowCardinality(String),
    device_type       LowCardinality(String),
    subscription_plan LowCardinality(String),
    converted         UInt8,
    signup_date       Date
) ENGINE = MergeTree()
ORDER BY (signup_date, user_id);
```

## Using with Filters

Compute associations for specific segments:

```sql
SELECT
    cramersVBiasCorrected(campaign_source, purchase_category) AS source_category_assoc
FROM orders
WHERE order_date >= today() - 30
  AND country = 'US';
```

## Limitations

- Both functions require categorical inputs. Continuous numeric variables should be binned first.
- Very high-cardinality columns (many unique values) can produce unreliable results.
- The functions operate over all rows in the group, so filter carefully to avoid mixing populations.

```sql
-- Bin a numeric column before using cramersV
SELECT
    cramersV(
        toString(intDiv(age, 10)),  -- age decade bucket
        subscription_tier
    ) AS age_tier_association
FROM users;
```

## Summary

`cramersV()` and `cramersVBiasCorrected()` provide SQL-native statistical association measurement for categorical variables in ClickHouse. Use `cramersV()` for large datasets and `cramersVBiasCorrected()` when working with smaller samples to avoid inflated association scores. These functions are powerful for feature selection, A/B test analysis, and exploratory data analysis.
