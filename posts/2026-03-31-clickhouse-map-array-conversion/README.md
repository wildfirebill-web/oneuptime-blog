# How to Convert Between Maps and Arrays in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Map Function, Array, Data Transformation

Description: Learn how to convert between Map and Array types in ClickHouse using mapKeys, mapValues, mapFromArrays, arrayZip, and CAST for round-trip data transformations.

---

ClickHouse provides a complete set of tools for converting between Map and Array representations. Understanding these conversions is essential when you need to apply array functions to map data, reconstruct Maps from aggregated arrays, or serialize maps for external systems. The conversions are lossless and composable, allowing you to move freely between representations depending on which operation is more convenient.

## Core Conversion Functions

The primary tools for converting Maps to Arrays and back are:

```text
mapKeys(m)                          -- Map -> Array of keys
mapValues(m)                        -- Map -> Array of values
arrayZip(keys, values)              -- two Arrays -> Array of tuples
mapFromArrays(keys_arr, vals_arr)   -- two Arrays -> Map
CAST(map AS Map(K, V))              -- type coercion
```

## Map to Arrays: Basic Extraction

Extract keys and values from a Map into separate arrays using `mapKeys()` and `mapValues()`.

```sql
SELECT
    mapKeys(map('x', 10, 'y', 20, 'z', 30))   AS keys,
    mapValues(map('x', 10, 'y', 20, 'z', 30)) AS values;
```

## Map to Array of Tuples

Use `arrayZip()` to produce a single array of `(key, value)` tuples, which is useful when you need to process entries as pairs rather than separate arrays.

```sql
SELECT
    arrayZip(
        mapKeys(map('a', 1, 'b', 2, 'c', 3)),
        mapValues(map('a', 1, 'b', 2, 'c', 3))
    ) AS kv_tuples;
```

The result is `[('a', 1), ('b', 2), ('c', 3)]`.

## Arrays Back to Map: mapFromArrays

Reconstruct a Map from two parallel arrays using `mapFromArrays()`.

```sql
SELECT mapFromArrays(['x', 'y', 'z'], [10, 20, 30]) AS reconstructed_map;
```

## Round-Trip Conversion

Verify that converting a Map to arrays and back produces an identical Map.

```sql
WITH source AS (
    SELECT map('host', 'web-01', 'env', 'prod', 'region', 'us-east') AS m
)
SELECT
    m AS original,
    mapFromArrays(mapKeys(m), mapValues(m)) AS round_tripped,
    m = mapFromArrays(mapKeys(m), mapValues(m)) AS is_equal
FROM source;
```

## Setting Up a Sample Table

Create a table storing request attributes as a Map to demonstrate real-world conversion patterns.

```sql
CREATE TABLE request_log
(
    request_id  UInt64,
    ts          DateTime,
    tags        Map(String, String)
)
ENGINE = MergeTree
ORDER BY (ts, request_id);

INSERT INTO request_log VALUES
(1, '2024-07-01 10:00:00', map('method', 'GET',  'status', '200', 'path', '/api/users', 'region', 'us-west')),
(2, '2024-07-01 10:01:00', map('method', 'POST', 'status', '201', 'path', '/api/orders', 'region', 'eu-central')),
(3, '2024-07-01 10:02:00', map('method', 'GET',  'status', '404', 'path', '/api/items/999', 'region', 'us-east'));
```

## Iterating Over Map Entries with ARRAY JOIN

To process each key-value pair as a separate row (useful for pivoting), use ARRAY JOIN on the tuple array produced by `arrayZip`.

```sql
SELECT
    request_id,
    kv.1 AS tag_key,
    kv.2 AS tag_value
FROM request_log
ARRAY JOIN arrayZip(mapKeys(tags), mapValues(tags)) AS kv
ORDER BY request_id, tag_key;
```

## Filtering Keys Then Rebuilding the Map

Extract a subset of keys using `arrayFilter()`, then reconstruct a Map from the filtered keys and their corresponding values.

```sql
SELECT
    request_id,
    mapFromArrays(
        arrayFilter(k -> k IN ('method', 'status'), mapKeys(tags)),
        arrayMap(k -> tags[k], arrayFilter(k -> k IN ('method', 'status'), mapKeys(tags)))
    ) AS http_tags_only
FROM request_log;
```

## Transforming Keys and Rebuilding

Use `arrayMap()` on the keys array to rename or reformat keys, then reconstruct the Map.

```sql
SELECT
    request_id,
    mapFromArrays(
        arrayMap(k -> concat('req.', k), mapKeys(tags)),
        mapValues(tags)
    ) AS prefixed_tags
FROM request_log;
```

## Converting a Map to a Sorted Array of Tuples

Sort the entries of a map by key before producing the tuple array, which is useful for deterministic serialization.

```sql
SELECT
    request_id,
    arraySort(
        t -> t.1,
        arrayZip(mapKeys(tags), mapValues(tags))
    ) AS sorted_kv_pairs
FROM request_log;
```

## Aggregating Tags Across Rows into a Single Map

Use `groupArray()` on both keys and values across all rows, then flatten and reconstruct using `mapFromArrays()` combined with `arrayFlatten()`.

```sql
SELECT
    mapFromArrays(
        arrayFlatten(groupArray(mapKeys(tags))),
        arrayFlatten(groupArray(mapValues(tags)))
    ) AS all_tags_combined
FROM request_log;
```

Note: this produces a map with the last value winning for any duplicate key. For merging with summation, use `sumMap()` instead.

## Summary

Converting between Maps and Arrays in ClickHouse involves `mapKeys()` and `mapValues()` to decompose a Map, `mapFromArrays()` to compose a Map from arrays, and `arrayZip()` to create paired tuple arrays for row-level iteration. These conversions are lossless and compose cleanly with ClickHouse's array function library. Use ARRAY JOIN on zipped pairs to pivot map entries into rows, use `arrayFilter()` and `arrayMap()` on the key/value arrays to transform before reconstruction, and use `arrayFlatten()` with `groupArray()` to merge maps across rows. Mastering these patterns enables full bidirectional movement between the Map and Array type systems.
