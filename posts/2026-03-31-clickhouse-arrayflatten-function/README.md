# How to Use arrayFlatten() Function in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Array Function, arrayFlatten, Nested Array

Description: Learn how arrayFlatten() collapses nested arrays (Array(Array(T))) into a flat Array(T) in ClickHouse, essential for working with grouped results and JSON-sourced nested arrays.

---

Working with JSON data or aggregating arrays often produces nested structures: arrays of arrays. Before you can apply standard array functions like `arrayDistinct`, `arraySort`, or `length`, you need to collapse these nested arrays into a flat list. ClickHouse's `arrayFlatten` function does exactly that, recursively collapsing one level of nesting.

## Function Signature

```text
arrayFlatten(arr_of_arrs) -> Array(T)
```

The input must be an `Array(Array(T))`. The output is a single `Array(T)` containing all the inner elements in order. `arrayFlatten` removes exactly one level of nesting per call - for deeper nesting, call it multiple times.

## Basic Usage

```sql
-- Flatten one level of nesting
SELECT arrayFlatten([[1, 2, 3], [4, 5], [6]]) AS flat;
-- Result: [1, 2, 3, 4, 5, 6]

-- Flatten arrays with empty sub-arrays
SELECT arrayFlatten([[1, 2], [], [3], []]) AS flat_with_empties;
-- Result: [1, 2, 3]

-- Flatten a single nested array
SELECT arrayFlatten([[10, 20, 30]]) AS single;
-- Result: [10, 20, 30]

-- Flatten string arrays
SELECT arrayFlatten([['hello', 'world'], ['foo', 'bar']]) AS flat_strings;
-- Result: ['hello', 'world', 'foo', 'bar']
```

## Flattening Grouped Aggregation Results

`groupArray` collects values into a single array per group. When combined with `groupArray` of arrays, or when aggregating array columns, you end up with nested arrays. `arrayFlatten` merges them:

```sql
CREATE TABLE user_tags
(
    user_id UInt32,
    tags Array(String)
) ENGINE = Memory;

INSERT INTO user_tags VALUES
    (1, ['sql', 'clickhouse', 'analytics']),
    (2, ['clickhouse', 'performance']),
    (3, ['sql', 'python', 'data']);

-- Collect all tag arrays across all users into one array of arrays
-- then flatten into a single tag list
SELECT
    arrayFlatten(groupArray(tags)) AS all_tags
FROM user_tags;
-- Result: ['sql','clickhouse','analytics','clickhouse','performance','sql','python','data']

-- Get unique tags across all users
SELECT
    arrayDistinct(arrayFlatten(groupArray(tags))) AS unique_tags
FROM user_tags;
-- Result: ['sql', 'clickhouse', 'analytics', 'performance', 'python', 'data']
```

## Flattening JSON-Sourced Nested Arrays

JSON documents often contain arrays of arrays - for example, a list of bounding boxes where each box is represented as a coordinate array. After parsing with `JSONExtract`, you get `Array(Array(Float32))`:

```sql
-- Simulate parsed JSON bounding boxes: [[x1,y1,x2,y2], [x1,y1,x2,y2], ...]
SELECT
    arrayFlatten(
        [
            [10.0, 20.0, 50.0, 60.0],
            [15.0, 25.0, 55.0, 65.0]
        ]
    ) AS all_coordinates;
-- Result: [10.0, 20.0, 50.0, 60.0, 15.0, 25.0, 55.0, 65.0]

-- Extract nested arrays from JSON then flatten
SELECT arrayFlatten(
    JSONExtract('{"groups":[[1,2,3],[4,5],[6,7,8]]}', 'groups', 'Array(Array(UInt32))')
) AS flat_groups;
-- Result: [1, 2, 3, 4, 5, 6, 7, 8]
```

## Combining Parallel Array Columns

When multiple array columns should be merged into a single list, wrap them in an outer array literal and flatten:

```sql
CREATE TABLE product_attributes
(
    product_id UInt32,
    colors Array(String),
    sizes Array(String)
) ENGINE = Memory;

INSERT INTO product_attributes VALUES
    (1, ['red', 'blue', 'green'], ['S', 'M', 'L', 'XL']),
    (2, ['black', 'white'], ['M', 'L']);

-- Combine colors and sizes into one flat attribute list
SELECT
    product_id,
    arrayFlatten([colors, sizes]) AS all_attributes
FROM product_attributes;
-- product 1: ['red','blue','green','S','M','L','XL']
-- product 2: ['black','white','M','L']
```

## Multi-Level Nesting - Apply Twice

For arrays nested more than two levels deep (`Array(Array(Array(T)))`), call `arrayFlatten` twice:

```sql
-- Two levels of nesting
SELECT arrayFlatten(arrayFlatten([[[1, 2], [3]], [[4, 5, 6]]])) AS double_flat;
-- First flatten: [[1,2,3],[4,5,6]]
-- Second flatten: [1,2,3,4,5,6]
```

## Counting Total Elements Across Nested Arrays

To count total elements in a nested array structure (the sum of lengths of all sub-arrays), flatten first then take the length:

```sql
SELECT
    user_id,
    tags AS original_tags_array,
    length(tags) AS num_tags
FROM user_tags;

-- Count total tag entries across all users
SELECT length(arrayFlatten(groupArray(tags))) AS total_tag_entries
FROM user_tags;
-- Result: 8 (counting duplicates across users)
```

## Summary

`arrayFlatten` collapses one level of array nesting, converting `Array(Array(T))` into `Array(T)`. It is essential whenever you aggregate array columns with `groupArray`, parse JSON arrays of arrays, or need to merge several parallel array columns into one. For deeper nesting, chain multiple `arrayFlatten` calls. After flattening, all standard array functions like `arrayDistinct`, `arraySort`, and `length` work on the combined flat array.
