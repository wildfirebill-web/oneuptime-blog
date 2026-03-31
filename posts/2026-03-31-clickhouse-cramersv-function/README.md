# How to Use cramersV() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, Aggregate Function, cramersV, Statistics, Association

Description: Learn how to use cramersV() and cramersVBiasCorrected() in ClickHouse to measure association strength between categorical variables using chi-squared statistics.

---

Cramer's V is a symmetric measure of association between two categorical variables, derived from the chi-squared statistic. It ranges from 0 (no association) to 1 (perfect association), making it easy to interpret. ClickHouse provides both `cramersV()` and a bias-corrected variant `cramersVBiasCorrected()` as aggregate functions, allowing categorical association analysis directly in SQL.

## What Cramer's V Measures

Cramer's V quantifies how strongly two categorical variables are related. It is based on the chi-squared test statistic but normalized to fall between 0 and 1 regardless of table size:

- **0.0** - no association between the variables
- **0.1 - 0.3** - weak association
- **0.3 - 0.7** - moderate association
- **0.7 - 1.0** - strong association
- **1.0** - perfect association

Unlike chi-squared alone, Cramer's V is comparable across different sample sizes and contingency table dimensions.

## Syntax

```sql
-- Standard Cramer's V
cramersV(x, y)

-- Bias-corrected version (recommended for small samples)
cramersVBiasCorrected(x, y)
```

Both accept two categorical columns and return a `Float64` in [0, 1].

## Creating Sample Data

```sql
CREATE TABLE user_survey
(
    user_id     UInt32,
    device_type String,
    plan_tier   String
)
ENGINE = MergeTree()
ORDER BY user_id;

INSERT INTO user_survey VALUES
    (1,  'mobile',  'free'),
    (2,  'mobile',  'free'),
    (3,  'mobile',  'pro'),
    (4,  'desktop', 'pro'),
    (5,  'desktop', 'enterprise'),
    (6,  'desktop', 'enterprise'),
    (7,  'tablet',  'free'),
    (8,  'tablet',  'pro'),
    (9,  'mobile',  'free'),
    (10, 'desktop', 'enterprise'),
    (11, 'mobile',  'pro'),
    (12, 'desktop', 'pro'),
    (13, 'tablet',  'free'),
    (14, 'mobile',  'free'),
    (15, 'desktop', 'enterprise');
```

## Basic Usage

```sql
SELECT cramersV(device_type, plan_tier)
FROM user_survey;
```

## Bias-Corrected Version

For small samples, the standard Cramer's V can be inflated. Use the bias-corrected version:

```sql
SELECT
    cramersV(device_type, plan_tier)              AS cramers_v,
    cramersVBiasCorrected(device_type, plan_tier) AS cramers_v_corrected
FROM user_survey;
```

The bias-corrected version adjusts for the number of rows and categories, providing a more accurate estimate especially when sample sizes are small.

## Comparing Multiple Variable Pairs

```sql
SELECT
    cramersV(device_type, plan_tier)       AS device_vs_plan,
    cramersV(device_type, signup_channel)  AS device_vs_channel,
    cramersV(plan_tier, signup_channel)    AS plan_vs_channel
FROM user_survey_extended;
```

## Association Heatmap Query

Generate a pairwise association matrix for all categorical columns of interest:

```sql
SELECT 'device_vs_plan'    AS pair, cramersV(device_type, plan_tier)      AS v FROM user_survey
UNION ALL
SELECT 'device_vs_region'  AS pair, cramersV(device_type, region)         AS v FROM user_survey
UNION ALL
SELECT 'plan_vs_region'    AS pair, cramersV(plan_tier, region)            AS v FROM user_survey
ORDER BY v DESC;
```

## Feature Selection for Machine Learning

Use Cramer's V to identify which categorical features are most associated with a target variable:

```sql
SELECT
    'device_type'     AS feature,
    cramersVBiasCorrected(device_type, churned) AS association
FROM user_churn_data

UNION ALL

SELECT
    'plan_tier'       AS feature,
    cramersVBiasCorrected(plan_tier, churned)   AS association
FROM user_churn_data

UNION ALL

SELECT
    'signup_channel'  AS feature,
    cramersVBiasCorrected(signup_channel, churned) AS association
FROM user_churn_data

ORDER BY association DESC;
```

Features with higher Cramer's V values are more associated with churn and may be more informative predictors.

## Interpreting Cramer's V

```sql
SELECT
    cramersVBiasCorrected(device_type, plan_tier) AS v,
    multiIf(
        v < 0.1,  'negligible',
        v < 0.3,  'weak',
        v < 0.7,  'moderate',
        'strong'
    ) AS association_strength
FROM user_survey;
```

## Relationship to Chi-Squared

Cramer's V is computed as:

```text
V = sqrt(chi2 / (n * (min(r, c) - 1)))
```

Where `chi2` is the chi-squared statistic, `n` is the sample size, `r` is the number of rows, and `c` is the number of columns in the contingency table. ClickHouse computes this automatically.

## Summary

`cramersV()` and `cramersVBiasCorrected()` in ClickHouse provide a normalized, easy-to-interpret measure of association between categorical variables based on the chi-squared statistic. The result ranges from 0 to 1 regardless of table dimensions or sample size, making it suitable for comparing associations across different variable pairs. Use the bias-corrected version for small samples, and use both functions for feature selection, survey analysis, and categorical data exploration.
