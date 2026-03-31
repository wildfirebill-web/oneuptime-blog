# How to Use JSONExtractKeys() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Json, Analytics, Query

Description: Learn how JSONExtractKeys() returns all top-level or nested keys from a JSON object as an Array(String) in ClickHouse, enabling dynamic schema inspection and key enumeration.

---

`JSONExtractKeys` returns an `Array(String)` containing all the keys of a JSON object at a given path. Unlike `JSONExtractKeysAndValues`, it returns only the key names without their associated values. This makes it useful for schema discovery, checking whether a specific key set is present, iterating over dynamic attribute names, and auditing payloads for unexpected fields.

## Basic Usage

```sql
-- Get all top-level keys from a JSON object
SELECT JSONExtractKeys('{"env": "prod", "region": "us-east", "version": "2.1"}') AS keys;
```

```text
keys
['env','region','version']
```

## Extracting Keys from a Column

```sql
-- List all top-level keys present in each event payload
SELECT
    event_id,
    JSONExtractKeys(payload) AS payload_keys
FROM events
LIMIT 10;
```

## Navigating to a Nested Object First

Pass additional string arguments to navigate to a nested object before listing its keys.

```sql
-- Get the keys of the nested 'settings' object
SELECT JSONExtractKeys(
    '{"user": "Alice", "settings": {"theme": "dark", "lang": "en", "tz": "UTC"}}',
    'settings'
) AS settings_keys;
```

```text
settings_keys
['theme','lang','tz']
```

## Checking Whether a Required Key Set is Present

```sql
-- Verify that all required fields are present
SELECT
    event_id,
    hasAll(
        JSONExtractKeys(payload),
        ['user_id', 'event_type', 'timestamp']
    ) AS has_required_fields
FROM events
WHERE hasAll(
    JSONExtractKeys(payload),
    ['user_id', 'event_type', 'timestamp']
) = 0
LIMIT 20;
```

## Counting Keys per Row

```sql
-- Find payloads with an unusually high number of top-level keys
SELECT
    event_id,
    length(JSONExtractKeys(payload)) AS key_count
FROM events
ORDER BY key_count DESC
LIMIT 10;
```

## Iterating Over Dynamic Keys

Combine `JSONExtractKeys` with `ARRAY JOIN` to get one row per key.

```sql
-- One row per key present in each payload
SELECT
    event_id,
    key_name
FROM events
ARRAY JOIN JSONExtractKeys(payload) AS key_name
ORDER BY event_id, key_name
LIMIT 30;
```

## Detecting Unexpected Keys

```sql
-- Find payloads that contain keys not in the expected set
SELECT
    event_id,
    arrayFilter(
        k -> NOT has(['user_id', 'event_type', 'timestamp', 'duration_ms'], k),
        JSONExtractKeys(payload)
    ) AS unexpected_keys
FROM events
WHERE length(arrayFilter(
    k -> NOT has(['user_id', 'event_type', 'timestamp', 'duration_ms'], k),
    JSONExtractKeys(payload)
)) > 0
LIMIT 10;
```

## Building a Key Frequency Report

```sql
-- Count how many rows contain each observed key across all payloads
SELECT
    key_name,
    count() AS rows_containing_key
FROM events
ARRAY JOIN JSONExtractKeys(payload) AS key_name
GROUP BY key_name
ORDER BY rows_containing_key DESC;
```

## Summary

`JSONExtractKeys` returns the key names of a JSON object as an `Array(String)`, optionally navigating a nested path first. Use it with `hasAll` to validate required fields, with `arrayFilter` to detect unexpected keys, with `ARRAY JOIN` to enumerate all keys across a dataset, and with `length` to measure object cardinality. It complements `JSONExtractKeysAndValues` when you only need key names rather than the associated values.
