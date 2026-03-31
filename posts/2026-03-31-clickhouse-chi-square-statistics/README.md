# How to Calculate Chi-Square Statistics in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Statistics, Chi-Square, Hypothesis Testing, Analytics

Description: Learn how to calculate chi-square statistics in ClickHouse to test independence between categorical variables and validate A/B test results.

---

## What Is the Chi-Square Test?

The chi-square test of independence checks whether two categorical variables are related or independent. It is widely used to validate A/B test results for conversion rates, check whether traffic distribution is balanced across variants, and identify unexpected patterns in categorical data.

## Building the Contingency Table

Suppose you want to test whether device type affects conversion:

```sql
SELECT
    device_type,
    countIf(converted = 1) AS converted,
    countIf(converted = 0) AS not_converted,
    count() AS total
FROM experiment_events
WHERE experiment_id = 'exp_mobile_redesign'
GROUP BY device_type;
```

## Computing Chi-Square Manually

ClickHouse does not have a single `chiSquareTest` function, but you can compute it in SQL. The formula is `sum((observed - expected)^2 / expected)`:

```sql
WITH
    grand_total AS (SELECT count() AS n FROM experiment_events),
    group_totals AS (
        SELECT
            device_type,
            countIf(converted = 1) AS obs_converted,
            countIf(converted = 0) AS obs_not_converted,
            count() AS row_total
        FROM experiment_events
        GROUP BY device_type
    ),
    col_totals AS (
        SELECT
            countIf(converted = 1) AS col_converted,
            countIf(converted = 0) AS col_not_converted
        FROM experiment_events
    )
SELECT
    sum(
        pow(g.obs_converted - (g.row_total * c.col_converted / gt.n), 2)
            / (g.row_total * c.col_converted / gt.n)
        + pow(g.obs_not_converted - (g.row_total * c.col_not_converted / gt.n), 2)
            / (g.row_total * c.col_not_converted / gt.n)
    ) AS chi_square_stat
FROM group_totals g
CROSS JOIN col_totals c
CROSS JOIN grand_total gt;
```

## Checking Traffic Balance

A simpler chi-square use case is testing whether traffic is evenly split between variants:

```sql
WITH
    expected AS (SELECT count() / 2.0 AS e FROM ab_events),
    counts AS (
        SELECT variant, count() AS observed
        FROM ab_events
        GROUP BY variant
    )
SELECT
    sum(pow(c.observed - e.e, 2) / e.e) AS chi_square,
    if(sum(pow(c.observed - e.e, 2) / e.e) > 3.841, 'imbalanced', 'balanced') AS verdict
FROM counts c CROSS JOIN expected e;
```

A chi-square value above 3.841 indicates imbalance at the 95% confidence level for 1 degree of freedom.

## Interpreting Critical Values

```text
Degrees of Freedom | 90% (p<0.10) | 95% (p<0.05) | 99% (p<0.01)
1                  | 2.706        | 3.841        | 6.635
2                  | 4.605        | 5.991        | 9.210
3                  | 6.251        | 7.815        | 11.345
```

## When to Use Chi-Square vs. Z-Test

```text
Chi-Square: categorical outcomes with multiple categories
Z-Test (proportionsZTest): binary outcomes comparing two proportions
T-Test (studentTTest): continuous outcomes comparing two means
```

## Summary

ClickHouse does not have a built-in chi-square function, but you can compute the statistic by constructing expected frequencies from marginal totals and summing the squared residuals. Use chi-square for testing independence between categorical variables and for validating traffic balance in experiments. Compare your result against standard critical value tables to determine significance.
