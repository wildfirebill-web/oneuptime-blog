# How to Use simpleJSONExtractRaw() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Json, Performance, Query

Description: Learn how simpleJSONExtractRaw() returns raw field values from flat JSON strings in ClickHouse using a fast heuristic parser, preserving the original JSON literal.

---

`simpleJSONExtractRaw` is the heuristic counterpart to `JSONExtractRaw`. It returns the raw JSON literal for a given key without fully parsing or validating the JSON structure. The result includes surrounding quotes for string values and preserves array or object literals as-is. It is faster than `JSONExtractRaw` on flat, well-formed inputs and is most useful when you need the verbatim value without type conversion and do not require nested path navigation.

## Basic Usage

```sql
-- Return the raw JSON literal for each key
SELECT
    simpleJSONExtractRaw('{"status": "ok", "code": 200, "active": true}', 'status') AS raw_status,
    simpleJSONExtractRaw('{"status": "ok", "code": 200, "active": true}', 'code')   AS raw_code,
    simpleJSONExtractRaw('{"status": "ok", "code": 200, "active": true}', 'active') AS raw_active;
```

```text
raw_status  raw_code  raw_active
"ok"        200       true
```

String values include surrounding double quotes; numbers and booleans do not.

## Comparing with JSONExtractRaw

```sql
-- Both return the same raw literal on flat well-formed JSON
SELECT
    simpleJSONExtractRaw('{"name": "Alice"}', 'name') AS simple_raw,
    JSONExtractRaw('{"name": "Alice"}', 'name')       AS full_raw;
```

```text
simple_raw  full_raw
"Alice"     "Alice"
```

The heuristic version skips full validation, so prefer the full version for complex or nested JSON.

## Extracting from a Table Column

```sql
-- Get the raw value of 'trace_id' from a log payload
SELECT
    log_id,
    simpleJSONExtractRaw(raw_log, 'trace_id') AS raw_trace_id
FROM logs
LIMIT 10;
```

## Using the Raw Value in a Comparison

Because string values include quotes, match against the quoted form.

```sql
-- Find logs where the raw status field equals the JSON string "error"
SELECT
    log_id,
    log_time
FROM logs
WHERE simpleJSONExtractRaw(raw_log, 'status') = '"error"'
ORDER BY log_time DESC
LIMIT 20;
```

## Forwarding a Raw Value to Another System

```sql
-- Store the raw value as-is for downstream systems that expect JSON literals
INSERT INTO raw_extracts (log_id, raw_payload_field)
SELECT
    log_id,
    simpleJSONExtractRaw(raw_log, 'context')
FROM logs
WHERE simpleJSONExtractRaw(raw_log, 'context') != '';
```

## Handling Absent Keys

An absent key returns an empty string.

```sql
-- Flag logs missing the 'request_id' field
SELECT
    log_id,
    simpleJSONExtractRaw(raw_log, 'request_id') = '' AS missing_request_id
FROM logs
WHERE simpleJSONExtractRaw(raw_log, 'request_id') = ''
LIMIT 10;
```

## When to Use the Full JSONExtractRaw Instead

Use `JSONExtractRaw` when:

- The value at the target key is a nested object or array and you need correct extraction
- The JSON contains escaped characters in string values
- You need to navigate nested paths with multiple arguments

```sql
-- Full path navigation, not supported by simpleJSONExtractRaw
SELECT JSONExtractRaw(payload, 'user', 'address') AS raw_address
FROM requests
LIMIT 5;
```

## Summary

`simpleJSONExtractRaw` returns the verbatim JSON literal for a top-level key in a flat JSON string, including quotes around string values. It is a faster alternative to `JSONExtractRaw` for high-throughput extraction from simple, known-good payloads. When you need nested path traversal or need to handle complex JSON, use `JSONExtractRaw` instead.
