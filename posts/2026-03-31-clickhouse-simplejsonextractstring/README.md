# How to Use simpleJSONExtractString() for Fast JSON Parsing in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, JSON, Performance, Query

Description: Learn how simpleJSONExtractString() offers a fast, heuristic-based alternative to JSONExtractString() for extracting string fields from simple, well-formed JSON in ClickHouse.

---

`simpleJSONExtractString` is a lightweight JSON string extractor that uses a heuristic parser instead of a full JSON parser. It is faster than `JSONExtractString` because it does not validate the full JSON structure, but it only works reliably on flat, well-formed JSON objects with simple string values. If your payloads are known to be well-structured and you need to extract high-volume string fields with minimal overhead, `simpleJSONExtractString` is a useful alternative.

## Basic Usage

```sql
-- Extract a string field from a simple flat JSON object
SELECT simpleJSONExtractString('{"env": "production", "region": "us-east-1"}', 'env') AS env;
```

```text
env
production
```

The return value is the unquoted string, just like `JSONExtractString`.

## Comparing with JSONExtractString

```sql
-- Both return the same result on well-formed flat JSON
SELECT
    simpleJSONExtractString('{"name": "Alice"}', 'name') AS simple_result,
    JSONExtractString('{"name": "Alice"}', 'name')       AS full_result;
```

```text
simple_result  full_result
Alice          Alice
```

The difference is performance: `simpleJSONExtractString` skips full validation, making it faster on simple inputs at scale.

## Extracting from a Table Column

```sql
-- Extract the service name from a high-volume log payload column
SELECT
    log_id,
    simpleJSONExtractString(raw_log, 'service')   AS service,
    simpleJSONExtractString(raw_log, 'log_level') AS log_level
FROM logs
LIMIT 20;
```

## Filtering with simpleJSONExtractString

```sql
-- Find all error logs for the auth service
SELECT
    log_id,
    log_time,
    simpleJSONExtractString(raw_log, 'message') AS message
FROM logs
WHERE
    simpleJSONExtractString(raw_log, 'service')   = 'auth'
    AND simpleJSONExtractString(raw_log, 'level') = 'error'
ORDER BY log_time DESC
LIMIT 50;
```

## Aggregating Extracted String Values

```sql
-- Count events per environment and region
SELECT
    simpleJSONExtractString(payload, 'env')    AS env,
    simpleJSONExtractString(payload, 'region') AS region,
    count()                                    AS event_count
FROM events
GROUP BY env, region
ORDER BY event_count DESC;
```

## Handling Absent Keys

`simpleJSONExtractString` returns an empty string when the key is not found, the same as `JSONExtractString`.

```sql
-- Check for rows where a field is absent (returns empty string)
SELECT
    event_id,
    simpleJSONExtractString(payload, 'correlation_id') AS correlation_id
FROM events
WHERE simpleJSONExtractString(payload, 'correlation_id') = ''
LIMIT 10;
```

## When to Prefer Full JSONExtract

`simpleJSONExtractString` is not suitable for:

- Nested objects (it does not support path traversal)
- JSON where string values contain escaped characters like `\n` or `\"`
- Heterogeneous or loosely structured payloads

```sql
-- For nested paths, always use the full JSONExtractString
SELECT JSONExtractString(payload, 'user', 'name') AS user_name
FROM events
LIMIT 5;
```

## Summary

`simpleJSONExtractString` is a fast heuristic extractor for flat, well-formed JSON string fields. It matches the return behavior of `JSONExtractString` for simple inputs but avoids full JSON parsing overhead. Reserve it for high-throughput pipelines where payloads are known to be simple. Use `JSONExtractString` for anything that involves nested paths, special characters in values, or complex JSON structures where correctness matters more than speed.
