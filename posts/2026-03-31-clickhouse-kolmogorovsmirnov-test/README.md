# How to Use kolmogorovSmirnovTest() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, Aggregate Function, Statistics, Distribution Test

Description: Learn how to use kolmogorovSmirnovTest() in ClickHouse to compare the full distributions of two groups, not just their means, for robust A/B testing.

---

The Kolmogorov-Smirnov (KS) test is a nonparametric test that compares the full cumulative distribution functions (CDFs) of two samples. Unlike a t-test or Mann-Whitney test, it is sensitive to differences in shape, spread, and location - not just the central tendency. ClickHouse's `kolmogorovSmirnovTest()` aggregate function brings this powerful distribution comparison directly into SQL.

## When to Use the KS Test

The KS test is appropriate when:
- You want to detect any distributional difference, not just a mean shift
- Your data is continuous and not necessarily normal
- You suspect the treatment changes the variance or tail behavior, not just the mean
- You are validating that a new data pipeline produces the same distribution as the old one

## Syntax

```sql
kolmogorovSmirnovTest(alternative)(sample_data, group_indicator)
```

Parameters:
- `alternative` - hypothesis direction: `'two-sided'`, `'greater'`, or `'less'`
- `sample_data` - numeric column of observations
- `group_indicator` - group label (0 or 1)

Returns a tuple `(statistic, p_value)`.

## Creating Sample Data

```sql
CREATE TABLE ks_test_data
(
    id      UInt32,
    variant UInt8,
    value   Float64
)
ENGINE = MergeTree()
ORDER BY id;

-- Control: roughly normal around 10
-- Treatment: roughly normal around 12 with wider spread
INSERT INTO ks_test_data
SELECT
    number                                          AS id,
    number % 2                                      AS variant,
    if(number % 2 = 0,
        10 + (rand() % 100) / 50.0 - 1.0,
        12 + (rand() % 200) / 50.0 - 2.0)          AS value
FROM numbers(1000);
```

## Running the Two-Sided Test

```sql
SELECT
    result.1 AS ks_statistic,
    result.2 AS p_value
FROM (
    SELECT kolmogorovSmirnovTest('two-sided')(value, variant) AS result
    FROM ks_test_data
);
```

The KS statistic is the maximum absolute difference between the two empirical CDFs. Values close to 0 indicate similar distributions; values close to 1 indicate very different distributions.

## Directional Tests

Test whether the treatment distribution is stochastically greater than the control:

```sql
SELECT
    result.1 AS ks_statistic,
    result.2 AS p_value
FROM (
    SELECT kolmogorovSmirnovTest('greater')(value, variant) AS result
    FROM ks_test_data
);
```

Use `'less'` to test whether the treatment tends to produce smaller values.

## Interpreting Results

| KS Statistic | Interpretation |
|---|---|
| ~0.0 | Distributions are nearly identical |
| 0.1 - 0.2 | Small distributional difference |
| 0.3 - 0.5 | Moderate difference |
| > 0.5 | Substantial distributional difference |

A low p-value (< 0.05) indicates the two samples are unlikely to come from the same underlying distribution.

## Goodness-of-Fit: Comparing Data Pipelines

Use the KS test to verify that a refactored data pipeline produces the same distribution:

```sql
SELECT
    result.1 AS ks_statistic,
    result.2 AS p_value,
    if(result.2 > 0.05,
        'distributions match',
        'distributions differ') AS verdict
FROM (
    SELECT kolmogorovSmirnovTest('two-sided')(latency_ms, pipeline_version) AS result
    FROM pipeline_comparison
);
```

## A/B Test: Revenue Distribution

A mean shift may not tell the full story - use KS to check if the treatment also changes variance or tail behavior:

```sql
CREATE TABLE revenue_ab
(
    user_id UInt32,
    variant UInt8,
    revenue Float64
)
ENGINE = MergeTree()
ORDER BY user_id;

INSERT INTO revenue_ab
SELECT
    number,
    number % 2,
    if(number % 2 = 0,
        5.0 + (rand() % 100) / 20.0,
        5.0 + (rand() % 300) / 20.0)   -- treatment has higher variance
FROM numbers(2000);

SELECT
    result.1 AS ks_statistic,
    result.2 AS p_value
FROM (
    SELECT kolmogorovSmirnovTest('two-sided')(revenue, variant) AS result
    FROM revenue_ab
);
-- Even if means are similar, a high KS statistic reveals the variance shift
```

## Comparing Multiple Segments

```sql
SELECT
    country,
    result.1 AS ks_statistic,
    result.2 AS p_value
FROM (
    SELECT
        country,
        kolmogorovSmirnovTest('two-sided')(value, variant) AS result
    FROM ks_test_data_with_country
    GROUP BY country
)
ORDER BY ks_statistic DESC;
```

## KS Test vs. Mean-Based Tests

```sql
-- Mean-based: may miss variance changes
SELECT welchTTest(0.95)(value, variant) FROM ks_test_data;

-- Distribution-based: detects shape changes too
SELECT kolmogorovSmirnovTest('two-sided')(value, variant) FROM ks_test_data;
```

Using both tests together provides a fuller picture: if the t-test is not significant but the KS test is, the treatment likely changed the variance or tail behavior rather than the mean.

## Summary

`kolmogorovSmirnovTest()` in ClickHouse tests whether two samples come from the same distribution by comparing their empirical CDFs. It detects differences in location, scale, and shape, making it more comprehensive than mean-only tests. It is particularly valuable for validating data pipeline changes, performing distribution-aware A/B testing, and catching tail-behavior shifts that a t-test would miss.
