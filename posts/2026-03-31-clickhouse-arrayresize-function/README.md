# How to Use arrayResize() Function in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Array Function, Machine Learning, arrayResize

Description: Learn how arrayResize() pads or truncates arrays to a fixed length in ClickHouse, ideal for normalizing array columns into fixed-length feature vectors for ML pipelines.

---

Machine learning pipelines, fixed-width feature stores, and schema normalization often require arrays of a consistent length. ClickHouse's `arrayResize` function solves this by either extending or truncating an array to an exact target size, optionally filling new positions with a default value. This lets you normalize ragged arrays into uniform feature vectors without leaving ClickHouse.

## Function Signature

```text
arrayResize(arr, size [, default])
```

- `arr` - the source array
- `size` - the desired output length (UInt or Int type)
- `default` - optional fill value for new positions; defaults to 0 for numbers and empty string for strings

## Basic Padding and Truncation

When `size` is greater than the array length, ClickHouse appends the default value to fill the gap. When `size` is smaller, the array is truncated to the first `size` elements.

```sql
-- Padding a short array to length 5 with zeros (default)
SELECT arrayResize([10, 20, 30], 5) AS padded;
-- Result: [10, 20, 30, 0, 0]

-- Truncating a longer array to length 3
SELECT arrayResize([10, 20, 30, 40, 50], 3) AS truncated;
-- Result: [10, 20, 30]

-- Same length - no change
SELECT arrayResize([1, 2, 3], 3) AS same;
-- Result: [1, 2, 3]
```

## Custom Default Fill Value

Providing a third argument controls what value fills new positions. This matters when 0 is a valid data point and you need a sentinel like -1 or NULL:

```sql
-- Fill with -1 as a sentinel value
SELECT arrayResize([1.0, 2.5], 5, -1.0) AS padded_sentinel;
-- Result: [1.0, 2.5, -1.0, -1.0, -1.0]

-- Fill with NULL (use Nullable array type)
SELECT arrayResize([1, 2], 4, NULL) AS padded_null;
-- Result: [1, 2, NULL, NULL]

-- Fill with a specific string
SELECT arrayResize(['cat', 'dog'], 5, 'unknown') AS padded_str;
-- Result: ['cat', 'dog', 'unknown', 'unknown', 'unknown']
```

## Normalizing Feature Vectors for ML

A common pattern in ML preprocessing is ensuring every row has the same number of features. Consider a table of user behavior events where the last N interactions are stored as arrays. Different users have different history lengths:

```sql
CREATE TABLE user_features
(
    user_id UInt32,
    last_scores Array(Float32)
) ENGINE = Memory;

INSERT INTO user_features VALUES
    (1, [0.9, 0.7, 0.8, 0.6, 0.5, 0.9]),
    (2, [0.4, 0.3]),
    (3, [0.8, 0.9, 0.7]),
    (4, []);

-- Normalize all feature vectors to exactly 5 elements
-- Truncate long arrays, pad short ones with 0.0
SELECT
    user_id,
    arrayResize(last_scores, 5, 0.0) AS feature_vector
FROM user_features;
-- user 1: [0.9, 0.7, 0.8, 0.6, 0.5]  (truncated from 6)
-- user 2: [0.4, 0.3, 0.0, 0.0, 0.0]  (padded from 2)
-- user 3: [0.8, 0.9, 0.7, 0.0, 0.0]  (padded from 3)
-- user 4: [0.0, 0.0, 0.0, 0.0, 0.0]  (padded from 0)
```

## Aligning Arrays for Element-Wise Operations

When computing element-wise arithmetic between two arrays of potentially different lengths, resize them to the same target length first to avoid errors:

```sql
-- Two arrays of different lengths - resize both before element-wise add
SELECT arrayMap(
    (a, b) -> a + b,
    arrayResize([1, 2, 3], 5, 0),
    arrayResize([10, 20], 5, 0)
) AS summed;
-- Result: [11, 22, 3, 0, 0]
```

## Building Fixed-Width Sliding Windows

`arrayResize` combined with `arraySlice` lets you extract fixed-length windows from time-series arrays. This is useful for feeding fixed-context windows into downstream models:

```sql
CREATE TABLE time_series
(
    sensor_id UInt32,
    readings Array(Float32)
) ENGINE = Memory;

INSERT INTO time_series VALUES
    (1, [1.1, 2.2, 3.3, 4.4, 5.5, 6.6, 7.7]),
    (2, [9.9, 8.8]);

-- Extract the last 5 readings, padding if the array is shorter than 5
SELECT
    sensor_id,
    arrayResize(
        arraySlice(readings, -5),  -- take up to last 5 elements
        5,
        0.0
    ) AS window_5
FROM time_series;
-- sensor 1: [3.3, 4.4, 5.5, 6.6, 7.7]
-- sensor 2: [9.9, 8.8, 0.0, 0.0, 0.0]
```

## Checking Array Lengths Before and After

It is good practice to verify that your resize is working as expected. Use `length` to inspect array sizes:

```sql
SELECT
    length(last_scores) AS original_len,
    length(arrayResize(last_scores, 5, 0.0)) AS resized_len,
    arrayResize(last_scores, 5, 0.0) AS resized
FROM user_features;
```

## Summary

`arrayResize` is the standard ClickHouse tool for enforcing fixed-length arrays. It truncates arrays that are too long, pads arrays that are too short with a configurable default, and leaves correctly-sized arrays unchanged. This makes it indispensable for ML feature engineering, element-wise operations between arrays, and any context where downstream code expects arrays of a uniform size.
