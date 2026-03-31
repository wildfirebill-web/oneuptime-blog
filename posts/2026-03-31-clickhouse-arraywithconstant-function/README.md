# How to Use arrayWithConstant() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Array Function, arrayWithConstant

Description: Learn how arrayWithConstant() generates arrays of any length filled with a constant value in ClickHouse, useful for padding, default vector initialization, and array construction.

---

Generating an array of repeated values is a simple but frequently needed operation: creating a zero-filled padding array, initializing a default feature vector, or building a mask array of ones. ClickHouse's `arrayWithConstant` function creates an array of a specified length where every element is the same constant value.

## Function Signature

```text
arrayWithConstant(n, val) -> Array(T)
```

- `n` - the length of the output array (UInt type)
- `val` - the value to fill every position

The type of the output array is inferred from the type of `val`.

## Basic Usage

```sql
-- Array of five zeros
SELECT arrayWithConstant(5, 0) AS zeros;
-- Result: [0, 0, 0, 0, 0]

-- Array of three strings
SELECT arrayWithConstant(3, 'unknown') AS placeholders;
-- Result: ['unknown', 'unknown', 'unknown']

-- Array of Float32 ones (useful for weight vectors)
SELECT arrayWithConstant(4, toFloat32(1.0)) AS ones;
-- Result: [1.0, 1.0, 1.0, 1.0]

-- Array of NULLs (use a Nullable type explicitly)
SELECT arrayWithConstant(3, CAST(NULL AS Nullable(Int32))) AS nulls;
-- Result: [NULL, NULL, NULL]

-- Zero-length array
SELECT arrayWithConstant(0, 99) AS empty;
-- Result: []
```

## Creating Padding Arrays for Fixed-Length Normalization

When combining arrays of different lengths, you may need to pad shorter arrays up to a target length. `arrayWithConstant` generates the padding chunk, which is then concatenated with `arrayConcat`:

```sql
CREATE TABLE feature_vectors
(
    item_id UInt32,
    features Array(Float32)
) ENGINE = Memory;

INSERT INTO feature_vectors VALUES
    (1, [0.1, 0.5, 0.3, 0.8, 0.2]),
    (2, [0.7, 0.4]),
    (3, []);

-- Pad all feature vectors to length 5 with 0.0
-- (arrayResize is more concise, but this shows arrayWithConstant explicitly)
SELECT
    item_id,
    arrayConcat(
        features,
        arrayWithConstant(
            greatest(0, toInt64(5) - toInt64(length(features))),
            toFloat32(0.0)
        )
    ) AS padded_to_5
FROM feature_vectors;
-- item 1: [0.1,0.5,0.3,0.8,0.2]  (no padding needed)
-- item 2: [0.7,0.4,0.0,0.0,0.0]  (padded by 3)
-- item 3: [0.0,0.0,0.0,0.0,0.0]  (padded by 5)
```

## Initializing Default Metric Arrays

When inserting a new record that needs a pre-sized array column initialized to a default state:

```sql
CREATE TABLE daily_counters
(
    user_id UInt32,
    -- 24 hourly buckets, initialized to 0
    hourly_hits Array(UInt32)
) ENGINE = MergeTree()
ORDER BY user_id;

-- Insert new user with zeroed-out 24-hour counter array
INSERT INTO daily_counters
SELECT
    1001 AS user_id,
    arrayWithConstant(24, toUInt32(0)) AS hourly_hits;

SELECT user_id, length(hourly_hits) AS num_buckets, hourly_hits[1] AS first_bucket
FROM daily_counters;
-- user 1001: 24 buckets, first_bucket = 0
```

## Generating Mask Arrays

A mask of 1s and 0s is useful when combined with `arrayMap` for conditional element selection. Generate a mask array of the same length as another array:

```sql
-- Create a ones mask the same length as a feature vector
SELECT
    item_id,
    features,
    arrayWithConstant(length(features), 1) AS mask
FROM feature_vectors;

-- Apply mask: element-wise multiply features by a custom mask
-- (mask out the 3rd feature by setting it to 0)
SELECT
    item_id,
    arrayMap(
        (f, m) -> f * m,
        features,
        arrayConcat(
            arrayWithConstant(2, toFloat32(1.0)),  -- keep first 2
            arrayWithConstant(1, toFloat32(0.0)),  -- zero out 3rd
            arrayWithConstant(2, toFloat32(1.0))   -- keep last 2
        )
    ) AS masked_features
FROM feature_vectors
WHERE item_id = 1;
-- Result: [0.1, 0.5, 0.0, 0.8, 0.2]
```

## Building Test Data Arrays

`arrayWithConstant` is excellent for generating test arrays of varying sizes to benchmark array functions:

```sql
-- Generate test rows with arrays of different sizes
SELECT
    n,
    arrayWithConstant(n, 1) AS arr,
    arrayReduce('sum', arrayWithConstant(n, 1)) AS sum_should_equal_n
FROM (
    SELECT arrayJoin([10, 100, 1000, 10000]) AS n
);
```

## Combining with range() for Indexed Arrays

Pair `arrayWithConstant` with `arrayZip` and `range` to create arrays of (index, constant_value) tuples:

```sql
-- Create indexed constant array: [(1,0),(2,0),(3,0),(4,0),(5,0)]
SELECT arrayZip(
    range(1, 6),
    arrayWithConstant(5, 0)
) AS indexed_zeros;
-- Result: [(1,0),(2,0),(3,0),(4,0),(5,0)]
```

## Generating Repeated Sequence Arrays

Combine `arrayWithConstant` to create repeating patterns via `arrayConcat` in a loop pattern:

```sql
-- Alternate 0,1 repeated 4 times = [0,1,0,1,0,1,0,1]
SELECT arrayFlatten(
    arrayMap(i -> [0, 1], range(4))
) AS alternating;
-- Result: [0,1,0,1,0,1,0,1]

-- Repeat [1,2,3] three times = [1,2,3,1,2,3,1,2,3]
SELECT arrayFlatten(
    arrayMap(i -> [1, 2, 3], range(3))
) AS repeated;
-- Result: [1,2,3,1,2,3,1,2,3]
```

## Summary

`arrayWithConstant(n, val)` creates an array of length `n` where every element equals `val`. It is the simplest way to generate zero-filled padding, ones masks, sentinel value arrays, and default initializers in ClickHouse. Pair it with `arrayConcat` for manual padding, `arrayMap` for masking operations, and `arrayZip` for indexed arrays. For the common case of padding an array to a fixed length, `arrayResize` is more concise, but `arrayWithConstant` is clearer when you specifically need to generate a standalone constant array.
