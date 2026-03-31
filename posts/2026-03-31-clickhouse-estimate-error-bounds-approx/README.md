# How to Estimate Error Bounds of Approximate Functions in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Approximate Query, Error Bound, Analytics, Statistics

Description: Learn how to understand and estimate the error bounds of approximate aggregate functions in ClickHouse to make informed trade-off decisions.

---

## Why Error Bounds Matter

Approximate query functions in ClickHouse trade accuracy for speed and memory. Understanding the error bounds of each function lets you choose the right tool for your use case - and avoid presenting misleading results to stakeholders.

## Approximate COUNT DISTINCT - uniq

`uniq` uses HyperLogLog and provides a typical error of around 2.2%:

```sql
SELECT uniq(user_id) AS approx_distinct_users
FROM events;

-- Compare with exact:
SELECT uniqExact(user_id) AS exact_distinct_users
FROM events;
```

The relative error for `uniq` is approximately `1.04 / sqrt(m)` where `m` is the number of registers (fixed internally). For most datasets, expect 1-3% error.

## Quantile Error - quantile vs quantileTDigest

Standard `quantile` uses reservoir sampling with a fixed-size reservoir:

```sql
SELECT
    quantile(0.99)(response_ms)        AS approx_p99,
    quantileExact(0.99)(response_ms)   AS exact_p99
FROM http_logs;
```

The relative error for reservoir sampling is bounded by `1 / sqrt(sample_size)`. For `quantileTDigest`, the error near the tails is proportional to `1/compression`:

```sql
-- With compression=100, relative error at P99 is ~1%
SELECT quantileTDigest(100)(0.99)(response_ms) AS p99
FROM http_logs;
```

## topK Error Bounds

`topK` uses the Space-Saving algorithm. It guarantees that any element appearing more than `total / k` times will be included, but counts can be overestimated:

```sql
SELECT topK(10)(event_type) FROM events;
```

The maximum overcount per item is bounded by `total_count / (k + 1)`.

## Visualizing Error in Practice

Run both exact and approximate versions and compute relative error:

```sql
SELECT
    approx,
    exact,
    abs(approx - exact) / exact AS relative_error
FROM (
    SELECT
        uniq(user_id)      AS approx,
        uniqExact(user_id) AS exact
    FROM events
    WHERE toDate(ts) = today()
);
```

## Sampling-Based Error

When using `SAMPLE` clauses, the error follows standard statistical sampling theory:

```sql
SELECT count() * 10 AS estimated_total
FROM http_logs SAMPLE 1/10;
```

For a sample of fraction `p`, the relative standard error is `1 / sqrt(p * N)` where `N` is the true count.

## Guidelines for Choosing Accuracy Targets

| Function | Typical Error | Memory |
|---|---|---|
| `uniq` | ~2% | Low |
| `uniqCombined` | ~0.1% | Medium |
| `quantile` | ~1-3% | Low |
| `quantileTDigest` | ~1% at P99 | Low |
| `topK(k)` | overcount bounded | Low |

```sql
-- Use uniqCombined when you need better accuracy than uniq
SELECT uniqCombined(user_id) AS accurate_distinct
FROM events;
```

## Summary

Understanding error bounds helps you select the right approximate function for each query. Use `uniqCombined` when count-distinct accuracy matters, `quantileTDigest` for tail latency with controlled error, and validate regularly by comparing approximate results against exact values on a sample of your data.
