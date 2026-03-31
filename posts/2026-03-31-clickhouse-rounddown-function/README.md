# How to Use roundDown() Function in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Math Function, Bucketing, Rounding

Description: Learn how roundDown() rounds a value down to the largest element in a custom array in ClickHouse, with examples for price tiers, custom bins, and discretizing values.

---

`roundDown()` rounds a value down to the nearest element in a user-provided sorted array. Unlike `floor()` which rounds to the nearest integer or `roundToExp2()` which uses power-of-2 boundaries, `roundDown()` lets you define arbitrary bucket boundaries. This makes it ideal for assigning values to non-uniform tiers: price brackets, age bands, user subscription levels, or any discrete categories with irregular spacing.

## Function Signature

```text
roundDown(x, array)
```

The `array` argument must be a sorted array of the same numeric type as `x`. The function returns the largest element in the array that is less than or equal to `x`. If `x` is smaller than all elements in the array, the function returns the first element. The array should be sorted in ascending order.

## Basic Usage

```sql
SELECT
    roundDown(5,    [1, 5, 10, 50, 100])  AS at_boundary,
    roundDown(7,    [1, 5, 10, 50, 100])  AS between_5_and_10,
    roundDown(0.5,  [1, 5, 10, 50, 100])  AS below_min,
    roundDown(200,  [1, 5, 10, 50, 100])  AS above_max;
```

`roundDown(7, ...)` returns 5 (the largest array element not exceeding 7). `roundDown(0.5, ...)` returns 1 (the smallest element, since x is below the minimum).

## Price Tier Assignment

Assign products to named price tiers using `roundDown()` on a set of tier boundaries.

```sql
CREATE TABLE product_catalog
(
    product_id   UInt64,
    product_name String,
    price        Float64
)
ENGINE = MergeTree
ORDER BY product_id;

INSERT INTO product_catalog VALUES
(1, 'Budget Widget',    4.99),
(2, 'Standard Widget', 14.99),
(3, 'Premium Widget',  34.99),
(4, 'Pro Widget',      74.99),
(5, 'Enterprise Pack', 249.99),
(6, 'Starter Kit',      9.99),
(7, 'Value Bundle',    24.99);
```

Use `roundDown()` to assign each product to its price tier.

```sql
SELECT
    product_name,
    price,
    roundDown(price, [0, 10, 25, 50, 100, 200]) AS tier_lower_bound,
    CASE roundDown(price, [0, 10, 25, 50, 100, 200])
        WHEN   0 THEN 'under $10'
        WHEN  10 THEN '$10 - $25'
        WHEN  25 THEN '$25 - $50'
        WHEN  50 THEN '$50 - $100'
        WHEN 100 THEN '$100 - $200'
        WHEN 200 THEN '$200+'
    END AS price_tier
FROM product_catalog
ORDER BY price;
```

## Latency Bucket Assignment

Group API latencies into non-uniform buckets that match common SLO threshold definitions.

```sql
CREATE TABLE latency_samples
(
    endpoint  String,
    latency_ms UInt32
)
ENGINE = MergeTree
ORDER BY endpoint;

INSERT INTO latency_samples VALUES
('/api/fast',   8), ('/api/fast',  15), ('/api/fast',  45),
('/api/fast', 105), ('/api/fast', 210), ('/api/fast', 550),
('/api/slow',  95), ('/api/slow', 285), ('/api/slow', 680),
('/api/slow', 1200);
```

```sql
SELECT
    endpoint,
    latency_ms,
    roundDown(latency_ms, [0, 10, 50, 100, 250, 500, 1000]) AS latency_bucket
FROM latency_samples
ORDER BY endpoint, latency_ms;
```

Aggregate to produce the latency distribution per endpoint.

```sql
SELECT
    endpoint,
    roundDown(latency_ms, [0, 10, 50, 100, 250, 500, 1000]) AS bucket,
    count() AS requests
FROM latency_samples
GROUP BY endpoint, bucket
ORDER BY endpoint, bucket;
```

## Age Demographic Bucketing

Assign customer ages to marketing demographic segments.

```sql
CREATE TABLE customers
(
    customer_id UInt64,
    age         UInt8
)
ENGINE = MergeTree
ORDER BY customer_id;

INSERT INTO customers VALUES
(1, 16), (2, 22), (3, 19), (4, 31), (5, 28),
(6, 45), (7, 52), (8, 67), (9, 72), (10, 38);
```

```sql
SELECT
    customer_id,
    age,
    roundDown(age, [13, 18, 25, 35, 45, 55, 65]) AS age_bucket,
    CASE roundDown(age, [13, 18, 25, 35, 45, 55, 65])
        WHEN 13 THEN '13-17'
        WHEN 18 THEN '18-24'
        WHEN 25 THEN '25-34'
        WHEN 35 THEN '35-44'
        WHEN 45 THEN '45-54'
        WHEN 55 THEN '55-64'
        WHEN 65 THEN '65+'
    END AS age_group
FROM customers
ORDER BY age;
```

## Discretizing Score Distributions

Convert continuous model scores into discrete grade bands with custom thresholds.

```sql
SELECT
    score,
    roundDown(score, [0, 60, 70, 80, 90]) AS grade_floor,
    CASE roundDown(score, [0, 60, 70, 80, 90])
        WHEN  0 THEN 'F'
        WHEN 60 THEN 'D'
        WHEN 70 THEN 'C'
        WHEN 80 THEN 'B'
        WHEN 90 THEN 'A'
    END AS grade
FROM (
    SELECT arrayJoin([45, 62, 71, 83, 91, 55, 78, 95]) AS score
)
ORDER BY score;
```

## Summary

`roundDown(x, array)` rounds a value down to the largest element in a user-defined sorted boundary array, making it the ideal function for irregular bin assignment. Use it for price tier labeling, latency bucket classification, age demographic segmentation, and any custom discretization scheme with non-uniform intervals. Pair with a CASE expression to convert the numeric bucket boundary into a human-readable label. The array must be sorted in ascending order and should cover the full expected range of input values for predictable results.
