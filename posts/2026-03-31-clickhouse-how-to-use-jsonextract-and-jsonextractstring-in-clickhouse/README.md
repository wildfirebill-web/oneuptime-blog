# How to Use JSONExtract() and JSONExtractString() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, JSON, JSONExtract, SQL, Data Parsing

Description: Learn how to use JSONExtract() and JSONExtractString() in ClickHouse to parse JSON stored in String columns and extract typed values efficiently.

---

## Overview

ClickHouse stores JSON as plain Strings in many schemas. The `JSONExtract*()` family of functions parses JSON at query time and extracts typed values. `JSONExtractString()` is the most common variant, extracting string fields, while `JSONExtract()` supports arbitrary types.

## JSONExtractString() - Extract a String Field

```sql
SELECT JSONExtractString('{"name": "Alice", "city": "NYC"}', 'name') AS name
-- Alice
```

Nested paths use dot notation as separate arguments:

```sql
SELECT JSONExtractString('{"user": {"name": "Bob", "role": "admin"}}', 'user', 'name') AS user_name
-- Bob
```

## JSONExtract() - Extract with Type Specification

`JSONExtract(json, type)` or `JSONExtract(json, path, type)` extracts and casts to the specified ClickHouse type:

```sql
SELECT JSONExtract('{"age": 30, "score": 99.5}', 'age', 'UInt32') AS age
-- 30

SELECT JSONExtract('{"age": 30, "score": 99.5}', 'score', 'Float64') AS score
-- 99.5
```

## Extracting from Table Columns

```sql
CREATE TABLE events
(
    event_id    UInt64,
    raw_payload String
)
ENGINE = MergeTree()
ORDER BY event_id;

INSERT INTO events VALUES
(1, '{"user_id": 1001, "action": "click", "page": "/home"}'),
(2, '{"user_id": 1002, "action": "view",  "page": "/docs"}');
```

Extract fields at query time:

```sql
SELECT
    event_id,
    JSONExtract(raw_payload, 'user_id', 'UInt64')  AS user_id,
    JSONExtractString(raw_payload, 'action')        AS action,
    JSONExtractString(raw_payload, 'page')          AS page
FROM events
```

## JSONExtractInt, JSONExtractFloat, JSONExtractBool

Convenience aliases for common types:

```sql
SELECT
    JSONExtractInt('{"count": 42}', 'count')        AS count,
    JSONExtractFloat('{"ratio": 0.75}', 'ratio')    AS ratio,
    JSONExtractBool('{"active": true}', 'active')   AS active
```

## Extracting Arrays

```sql
SELECT JSONExtract('{"tags": ["sql", "clickhouse", "olap"]}', 'tags', 'Array(String)') AS tags
-- ['sql', 'clickhouse', 'olap']
```

Then use array functions on the result:

```sql
SELECT
    event_id,
    JSONExtract(raw_payload, 'tags', 'Array(String)') AS tags,
    has(JSONExtract(raw_payload, 'tags', 'Array(String)'), 'clickhouse') AS is_ch
FROM events
```

## Extracting Nested Objects

```sql
SELECT
    JSONExtract(
        '{"metadata": {"source": "api", "version": 2}}',
        'metadata',
        'Map(String, String)'
    ) AS metadata_map
```

Or use chained path notation:

```sql
SELECT JSONExtractString('{"a": {"b": {"c": "deep"}}}', 'a', 'b', 'c') AS deep_value
-- deep
```

## Handling Missing Keys

Missing keys return empty string for `JSONExtractString` and 0 for numeric types. Use `JSONHas()` to check key existence:

```sql
SELECT
    JSONHas('{"a": 1, "b": 2}', 'a') AS has_a,
    JSONHas('{"a": 1, "b": 2}', 'c') AS has_c
-- 1, 0
```

## Performance Tip - Use Materialized Columns

For frequently queried JSON fields, use materialized columns to avoid repeated parsing:

```sql
ALTER TABLE events
    ADD COLUMN user_id  UInt64 MATERIALIZED JSONExtract(raw_payload, 'user_id', 'UInt64'),
    ADD COLUMN action   String MATERIALIZED JSONExtractString(raw_payload, 'action');

ALTER TABLE events MATERIALIZE COLUMN user_id;
ALTER TABLE events MATERIALIZE COLUMN action;
```

## Summary

`JSONExtractString()` and `JSONExtract()` parse JSON stored in String columns at query time. Use typed variants (`JSONExtractInt`, `JSONExtractFloat`) for convenience, extract arrays with `Array(T)` type hints, and check key existence with `JSONHas()`. For hot paths, materialize frequently extracted fields into dedicated columns to avoid re-parsing on every query.
