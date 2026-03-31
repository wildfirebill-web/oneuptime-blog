# How to Use JSONHas() and JSONLength() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, JSON, Analytics, Query

Description: Learn how JSONHas() checks for field existence and JSONLength() counts elements in JSON objects and arrays in ClickHouse, with practical examples for validation and filtering.

---

`JSONHas` and `JSONLength` are two complementary inspection functions in ClickHouse. `JSONHas` returns `1` if a given key or path exists in a JSON string and `0` otherwise. `JSONLength` returns the number of keys in a JSON object or the number of elements in a JSON array. Together they let you validate JSON structure, guard against missing fields, and filter rows based on the shape of their JSON payloads.

## JSONHas: Checking for a Field

```sql
-- Check whether individual keys are present
SELECT
    JSONHas('{"name": "Alice", "age": 30}', 'name')    AS has_name,
    JSONHas('{"name": "Alice", "age": 30}', 'address') AS has_address;
```

```text
has_name  has_address
1         0
```

## JSONLength on an Object

```sql
-- Count the number of top-level keys in the object
SELECT JSONLength('{"a": 1, "b": 2, "c": 3}');
```

```text
3
```

## JSONLength on an Array

```sql
-- Count elements in a JSON array
SELECT JSONLength('["x", "y", "z"]');
```

```text
3
```

## Filtering Rows That Have a Required Field

```sql
-- Only process events that carry a session_id
SELECT
    event_id,
    JSONExtractString(payload, 'session_id') AS session_id,
    event_time
FROM events
WHERE JSONHas(payload, 'session_id') = 1
ORDER BY event_time DESC
LIMIT 20;
```

## Validating Minimum Object Completeness

```sql
-- Find user profiles missing required fields
SELECT
    user_id,
    JSONHas(profile, 'email')   AS has_email,
    JSONHas(profile, 'country') AS has_country
FROM users
WHERE
    JSONHas(profile, 'email') = 0
    OR JSONHas(profile, 'country') = 0
ORDER BY user_id;
```

## Filtering by Array Length

```sql
-- Orders with at least two line items
SELECT
    order_id,
    JSONLength(order_json, 'items') AS item_count
FROM orders
WHERE JSONLength(order_json, 'items') >= 2
ORDER BY item_count DESC
LIMIT 20;
```

Pass the field name as a second argument to `JSONLength` to navigate directly to an array or nested object.

## Checking Nested Key Existence

```sql
-- Check whether a deeply nested key exists
SELECT
    event_id,
    JSONHas(payload, 'context', 'device', 'os') AS has_os
FROM events
WHERE JSONHas(payload, 'context', 'device', 'os') = 1
LIMIT 10;
```

## Combining Both Functions for Schema Audit

```sql
-- Summarize JSON schema coverage across all rows
SELECT
    countIf(JSONHas(payload, 'user_id'))    AS rows_with_user_id,
    countIf(JSONHas(payload, 'session_id')) AS rows_with_session_id,
    avg(JSONLength(payload))                AS avg_top_level_keys,
    count()                                 AS total_rows
FROM events;
```

## Summary

`JSONHas` is the correct way to test for key presence in ClickHouse JSON processing because `JSONExtract*` functions return type defaults (`0`, empty string) for missing keys, making absence indistinguishable from genuine zero or empty values. `JSONLength` gives you the element count for arrays and objects, enabling size-based filters and completeness checks. Use both together to build defensive queries that validate JSON structure before extracting values.
