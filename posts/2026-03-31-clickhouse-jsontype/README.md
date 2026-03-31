# How to Use JSONType() Function in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, JSON, Analytics, Query

Description: Learn how JSONType() identifies the JSON type of a field in ClickHouse, returning values like String, Int64, Array, or Object to help with dynamic schema inspection.

---

`JSONType` returns a `String` describing the JSON type of a value at a given path within a JSON string. The possible return values are `String`, `Int64`, `UInt64`, `Float64`, `Bool`, `Array`, `Object`, `Null`, and an empty string when the key does not exist. This function is useful for inspecting the actual type before extracting, auditing mixed-type fields, and debugging payloads that do not conform to an expected schema.

## Basic Usage

```sql
-- Identify the JSON type of each field
SELECT
    JSONType('{"id": 1, "name": "Alice", "scores": [9, 8], "meta": {}}', 'id')     AS id_type,
    JSONType('{"id": 1, "name": "Alice", "scores": [9, 8], "meta": {}}', 'name')   AS name_type,
    JSONType('{"id": 1, "name": "Alice", "scores": [9, 8], "meta": {}}', 'scores') AS scores_type,
    JSONType('{"id": 1, "name": "Alice", "scores": [9, 8], "meta": {}}', 'meta')   AS meta_type;
```

```text
id_type  name_type  scores_type  meta_type
Int64    String     Array        Object
```

## Checking Types Before Extraction

```sql
-- Only extract as float if the value is actually numeric
SELECT
    event_id,
    JSONType(payload, 'amount')                                           AS amount_type,
    if(
        JSONType(payload, 'amount') IN ('Int64', 'UInt64', 'Float64'),
        JSONExtractFloat(payload, 'amount'),
        0.0
    ) AS safe_amount
FROM events
LIMIT 10;
```

## Auditing a Column for Type Consistency

```sql
-- Count how many rows have each type for the 'value' field
SELECT
    JSONType(payload, 'value') AS value_type,
    count()                    AS row_count
FROM events
GROUP BY value_type
ORDER BY row_count DESC;
```

This reveals whether a field is ever `Null`, sometimes a `String` and sometimes a number, or contains nested objects when only scalars are expected.

## Detecting Missing vs Null

An empty string means the key was absent; `'Null'` means the key exists with a JSON `null` value.

```sql
-- Distinguish missing keys from explicit null values
SELECT
    event_id,
    JSONType(payload, 'cancelled_at') AS cancelled_at_type
FROM events
WHERE JSONType(payload, 'cancelled_at') IN ('', 'Null')
LIMIT 10;
```

## Filtering Rows That Have an Array Field

```sql
-- Only process rows where 'tags' is actually a JSON array
SELECT
    post_id,
    JSONExtractArrayRaw(metadata, 'tags') AS tags
FROM posts
WHERE JSONType(metadata, 'tags') = 'Array'
LIMIT 10;
```

## Checking Nested Field Types

```sql
-- Navigate into a nested object to check the type of an inner field
SELECT
    user_id,
    JSONType(profile, 'address', 'zip') AS zip_type
FROM users
LIMIT 10;
```

## Building a Schema Summary

```sql
-- Summarize the types of every top-level key across all payloads
SELECT
    kv.1                           AS field_name,
    JSONType(payload, kv.1)        AS field_type,
    count()                        AS occurrences
FROM events
ARRAY JOIN JSONExtractKeysAndValues(payload, String) AS kv
GROUP BY field_name, field_type
ORDER BY field_name, occurrences DESC;
```

## Summary

`JSONType` exposes the runtime JSON type of a field, returning one of `String`, `Int64`, `UInt64`, `Float64`, `Bool`, `Array`, `Object`, `Null`, or an empty string for absent keys. Use it to audit columns for unexpected type variance, guard extraction logic with type checks, and distinguish JSON `null` from a missing field. It is especially helpful when ingesting data from external systems that do not enforce a strict schema.
