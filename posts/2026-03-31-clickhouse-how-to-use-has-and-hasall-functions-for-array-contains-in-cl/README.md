# How to Use has() and hasAll() Functions for Array Contains in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Arrays, has, hasAll, Array Contains, Sql

Description: Learn how to use has() and hasAll() in ClickHouse to check if arrays contain specific elements or sets of elements for array filtering.

---

## Overview

ClickHouse provides `has()` and `hasAll()` for array membership testing. `has(arr, elem)` checks whether a single element is present in an array. `hasAll(arr, subset)` checks whether all elements of a second array are contained within the first.

## has() - Single Element Membership

```sql
SELECT has(['apple', 'banana', 'cherry'], 'banana') AS found
-- 1 (true)

SELECT has(['apple', 'banana', 'cherry'], 'grape') AS found
-- 0 (false)
```

With numeric arrays:

```sql
SELECT has([1, 2, 3, 4, 5], 3) AS contains_3
-- 1
```

## hasAll() - Subset Containment

`hasAll(arr, subset)` returns 1 if every element in the second array appears in the first:

```sql
SELECT hasAll(['a', 'b', 'c', 'd'], ['a', 'c']) AS all_present
-- 1

SELECT hasAll(['a', 'b', 'c'], ['a', 'x']) AS all_present
-- 0 (x is missing)
```

## hasAny() - Intersection Check

`hasAny(arr, subset)` returns 1 if at least one element from the second array is found in the first:

```sql
SELECT hasAny(['a', 'b', 'c'], ['x', 'b', 'z']) AS any_match
-- 1
```

## Filtering Rows with has()

Use `has()` in WHERE clauses to filter rows where an array column contains a specific value:

```sql
CREATE TABLE articles
(
    article_id  UInt64,
    tags        Array(String)
)
ENGINE = MergeTree()
ORDER BY article_id;

INSERT INTO articles VALUES
(1, ['sql', 'database', 'clickhouse']),
(2, ['python', 'machine-learning']),
(3, ['clickhouse', 'performance']);
```

Find all articles tagged `clickhouse`:

```sql
SELECT article_id, tags
FROM articles
WHERE has(tags, 'clickhouse')
```

Output:

```text
article_id | tags
1          | ['sql', 'database', 'clickhouse']
3          | ['clickhouse', 'performance']
```

## Filtering with hasAll()

Find articles that have both `clickhouse` and `performance` tags:

```sql
SELECT article_id, tags
FROM articles
WHERE hasAll(tags, ['clickhouse', 'performance'])
```

## Using has() with Numeric Arrays

```sql
SELECT user_id, purchased_products
FROM orders
WHERE has(purchased_products, 42)
-- Find all orders that include product ID 42
```

## Counting Matches

Count how many rows contain a specific tag:

```sql
SELECT
    tag,
    countIf(has(tags, tag)) AS article_count
FROM articles
CROSS JOIN (SELECT arrayJoin(['sql', 'clickhouse', 'python']) AS tag)
GROUP BY tag
```

## Combining has() and hasAny() for Flexible Tag Filters

Find articles matching any of a list of tags (multi-tag OR filter):

```sql
SELECT article_id, tags
FROM articles
WHERE hasAny(tags, ['sql', 'machine-learning'])
```

## Performance Note

For high-cardinality arrays, `has()` performs a linear scan. For better performance on large datasets, consider:
- Keeping arrays small
- Using Bloom filter indexes on Array columns
- Using a separate tag mapping table with a join

```sql
-- Add a Bloom filter index on an array column
ALTER TABLE articles ADD INDEX tags_bloom_filter tags TYPE bloom_filter GRANULARITY 1;
ALTER TABLE articles MATERIALIZE INDEX tags_bloom_filter;
```

## Summary

`has()` checks single element membership in a ClickHouse array, `hasAll()` verifies full subset containment, and `hasAny()` checks for any intersection. These functions are central to tag-based filtering, feature flag lookups, and multi-label classification queries on Array columns.
