# How to Use arrayElement() and Indexing in ClickHouse Arrays

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Arrays, arrayElement, Indexing, Sql

Description: Learn how to use arrayElement() and bracket notation to access array elements by index in ClickHouse, including negative indexing and safe access patterns.

---

## Overview

ClickHouse arrays are 1-indexed (not 0-indexed like most programming languages). `arrayElement(arr, n)` retrieves the element at position `n`. The bracket notation `arr[n]` is an equivalent shorthand. ClickHouse also supports negative indices for accessing elements from the end of the array.

## Basic Usage

```sql
SELECT
    arrayElement(['a', 'b', 'c', 'd'], 1) AS first,
    arrayElement(['a', 'b', 'c', 'd'], 2) AS second,
    arrayElement(['a', 'b', 'c', 'd'], 4) AS last
```

Output:

```text
first | second | last
a     | b      | d
```

## Bracket Notation

The bracket shorthand is identical to `arrayElement()`:

```sql
SELECT
    ['x', 'y', 'z'][1] AS first,
    ['x', 'y', 'z'][3] AS third
```

## Negative Indexing

Negative indices count from the end of the array:

```sql
SELECT
    ['a', 'b', 'c', 'd'][-1] AS last_element,
    ['a', 'b', 'c', 'd'][-2] AS second_to_last
```

Output:

```text
last_element | second_to_last
d            | c
```

## Out-of-Bounds Access

Accessing an index beyond the array length returns the default value for the element type (0 for numbers, empty string for String):

```sql
SELECT
    [1, 2, 3][10] AS out_of_bounds
-- Returns: 0
```

To detect out-of-bounds access safely, check the array length first:

```sql
SELECT
    arr,
    length(arr) AS arr_len,
    if(length(arr) >= 2, arr[2], NULL) AS safe_second
FROM (SELECT [1] AS arr UNION ALL SELECT [1, 2, 3] AS arr)
```

## Accessing Array Columns in Tables

Given a table with Array columns:

```sql
CREATE TABLE event_log
(
    event_id  UInt64,
    tags      Array(String),
    metrics   Array(Float64)
)
ENGINE = MergeTree()
ORDER BY event_id;

INSERT INTO event_log VALUES
(1, ['error', 'critical', 'timeout'], [0.95, 1.2, 0.33]),
(2, ['info', 'start'], [0.1, 0.05]);
```

Access specific elements:

```sql
SELECT
    event_id,
    tags[1]    AS first_tag,
    tags[-1]   AS last_tag,
    metrics[1] AS first_metric
FROM event_log
```

## Using arrayElement with Dynamic Index

Pass a column or expression as the index:

```sql
SELECT
    event_id,
    arrayElement(tags, toUInt64(length(tags))) AS last_tag_dynamic
FROM event_log
```

## Extracting First N Elements

Use `arraySlice()` together with indexing to extract sub-arrays:

```sql
SELECT
    arraySlice(['a','b','c','d','e'], 1, 3) AS first_three
-- ['a', 'b', 'c']
```

## Combining with arrayMap and arrayFilter

You can index into nested arrays after a map operation:

```sql
SELECT
    arrayMap(arr -> arr[1], [[1,2],[3,4],[5,6]]) AS first_elements
-- [1, 3, 5]
```

## Real-World Use Case - Top Path Segment

Extract the first path segment from URL arrays stored in ClickHouse:

```sql
SELECT
    user_id,
    path_segments[1] AS top_level_path
FROM navigation_events
WHERE length(path_segments) > 0
```

## Summary

ClickHouse array indexing is 1-based and supports negative indices for accessing from the end. `arrayElement(arr, n)` and the `arr[n]` bracket notation are equivalent. Out-of-bounds access returns type defaults rather than errors, so validate array length when strict null safety is needed.
