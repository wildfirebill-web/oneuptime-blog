# How to Work with JSON Arrays in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, JSON, Array, Analytics, Query

Description: Learn how to work with JSON arrays in ClickHouse using JSONExtractArrayRaw, arrayJoin, and typed extraction functions to process and aggregate array-valued fields efficiently.

---

JSON arrays inside string columns are a common pattern when ingesting event data, order line items, tags, or log entries. ClickHouse provides several tools for working with these arrays: `JSONExtractArrayRaw` converts a JSON array into a native `Array(String)` of raw JSON elements, typed variants like `JSONExtract(col, 'field', 'Array(String)')` parse arrays of scalars directly, and `arrayJoin` or `ARRAY JOIN` flatten array rows for per-element processing. This guide covers the full workflow from extraction to aggregation.

## Extracting an Array of Strings

```sql
-- Parse a JSON string array into Array(String)
SELECT JSONExtract('{"tags": ["analytics", "clickhouse", "json"]}', 'tags', 'Array(String)') AS tags;
```

```text
tags
['analytics','clickhouse','json']
```

## Extracting an Array of Integers

```sql
-- Parse a JSON integer array into Array(Int64)
SELECT JSONExtract('{"scores": [10, 20, 30]}', 'scores', 'Array(Int64)') AS scores;
```

```text
scores
[10,20,30]
```

## Flattening Array Elements into Rows with arrayJoin

```sql
-- One row per tag extracted from a posts table
SELECT
    post_id,
    arrayJoin(
        JSONExtract(metadata, 'tags', 'Array(String)')
    ) AS tag
FROM posts
LIMIT 20;
```

## Using ARRAY JOIN for Multiple Arrays

```sql
-- Join the flattened array with the outer row using ARRAY JOIN syntax
SELECT
    post_id,
    tag
FROM posts
ARRAY JOIN JSONExtract(metadata, 'tags', 'Array(String)') AS tag
WHERE tag != ''
LIMIT 20;
```

## Counting Occurrences of Each Tag

```sql
-- Most popular tags across all posts
SELECT
    tag,
    count() AS usage_count
FROM posts
ARRAY JOIN JSONExtract(metadata, 'tags', 'Array(String)') AS tag
GROUP BY tag
ORDER BY usage_count DESC
LIMIT 20;
```

## Working with Arrays of Objects

For arrays of JSON objects, use `JSONExtractArrayRaw` then parse each element.

```sql
-- Extract item names from an order items array
SELECT
    order_id,
    JSONExtractString(item, 'name')   AS item_name,
    JSONExtractFloat(item, 'price')   AS item_price,
    JSONExtractInt(item, 'qty')       AS quantity
FROM orders
ARRAY JOIN JSONExtractArrayRaw(order_json, 'items') AS item
ORDER BY order_id;
```

## Checking Whether an Array Contains a Value

```sql
-- Find posts tagged with 'clickhouse'
SELECT
    post_id,
    post_title
FROM posts
WHERE has(
    JSONExtract(metadata, 'tags', 'Array(String)'),
    'clickhouse'
);
```

## Computing Array Statistics

```sql
-- Average number of tags per post, max and min
SELECT
    avg(length(JSONExtract(metadata, 'tags', 'Array(String)'))) AS avg_tags,
    max(length(JSONExtract(metadata, 'tags', 'Array(String)'))) AS max_tags,
    min(length(JSONExtract(metadata, 'tags', 'Array(String)'))) AS min_tags
FROM posts;
```

## Sorting and Deduplicating Array Elements

```sql
-- Unique sorted tags for each post
SELECT
    post_id,
    arraySort(arrayDistinct(
        JSONExtract(metadata, 'tags', 'Array(String)')
    )) AS unique_sorted_tags
FROM posts
LIMIT 10;
```

## Summary

Work with JSON arrays in ClickHouse by choosing the right extraction function based on element type: `JSONExtract` with an `Array(T)` type argument for homogeneous scalar arrays, and `JSONExtractArrayRaw` for arrays of objects or mixed types. Use `ARRAY JOIN` or `arrayJoin` to flatten arrays into rows, then apply aggregate functions as if each element were its own row. Array functions like `has`, `arrayDistinct`, `arraySort`, and `length` work directly on the extracted native arrays.
