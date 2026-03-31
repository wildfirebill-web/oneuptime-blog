# How to Use simpleJSONHas() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Json, Performance, Query

Description: Learn how simpleJSONHas() quickly checks for field existence in flat JSON strings using a heuristic parser in ClickHouse, returning 1 when the key is present and 0 otherwise.

---

`simpleJSONHas` checks whether a given key exists in a flat JSON object using a heuristic parser. It returns `1` if the key is present and `0` if it is not. Like the other `simpleJSON*` functions, it skips full JSON validation and is therefore faster than `JSONHas` on high-throughput workloads where the input JSON is known to be well-formed and flat. It does not support nested path traversal.

## Basic Usage

```sql
-- Check for key presence using the heuristic parser
SELECT
    simpleJSONHas('{"host": "web-01", "env": "prod"}', 'host') AS has_host,
    simpleJSONHas('{"host": "web-01", "env": "prod"}', 'port') AS has_port;
```

```text
has_host  has_port
1         0
```

## Comparing with JSONHas

```sql
-- Both return the same result on flat well-formed JSON
SELECT
    simpleJSONHas('{"status": "ok"}', 'status') AS simple_has,
    JSONHas('{"status": "ok"}', 'status')        AS full_has;
```

```text
simple_has  full_has
1           1
```

On flat, valid JSON the results are identical; `simpleJSONHas` just does less work to get there.

## Filtering Rows That Have a Required Field

```sql
-- Only process log rows that carry a trace_id
SELECT
    log_id,
    log_time,
    simpleJSONExtractString(raw_log, 'trace_id') AS trace_id
FROM logs
WHERE simpleJSONHas(raw_log, 'trace_id') = 1
ORDER BY log_time DESC
LIMIT 20;
```

## Finding Rows Missing a Field

```sql
-- Audit: which events are missing the required 'user_id' field?
SELECT
    event_id,
    event_time
FROM events
WHERE simpleJSONHas(payload, 'user_id') = 0
ORDER BY event_time DESC
LIMIT 50;
```

## Counting Field Coverage Across a Dataset

```sql
-- What percentage of log rows have each optional field?
SELECT
    countIf(simpleJSONHas(raw_log, 'session_id'))     AS has_session_id,
    countIf(simpleJSONHas(raw_log, 'correlation_id')) AS has_correlation_id,
    countIf(simpleJSONHas(raw_log, 'user_agent'))     AS has_user_agent,
    count()                                           AS total_rows
FROM logs;
```

## Guarding an Extraction

```sql
-- Safe extraction: only return amount when the field is present
SELECT
    order_id,
    if(
        simpleJSONHas(order_json, 'discount'),
        simpleJSONExtractFloat(order_json, 'discount'),
        0.0
    ) AS discount
FROM orders
LIMIT 10;
```

## When to Use JSONHas Instead

Use the full `JSONHas` when:

- The key is inside a nested object (requires path arguments)
- The JSON may contain unusual whitespace or escaped characters that confuse the heuristic parser

```sql
-- Full JSONHas with a nested path
SELECT JSONHas(payload, 'user', 'address', 'city') AS has_city
FROM requests
LIMIT 5;
```

## Summary

`simpleJSONHas` is a fast, heuristic check for key existence in flat, well-formed JSON strings. It returns the same `1`/`0` as `JSONHas` for simple inputs but avoids full parsing overhead. Use it in high-throughput pipelines where JSON structure is predictable and flat. For nested path checks, fall back to `JSONHas` with multiple path arguments.
