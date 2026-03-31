# How to Store and Query JSON Data Types in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, JSON, JSONExtract, Semi-Structured Data, Analytics

Description: Learn the different approaches to storing and querying JSON data in ClickHouse, from string-based JSON functions to the native Object type.

---

## Approaches to Storing JSON in ClickHouse

ClickHouse provides several approaches for working with JSON data:

1. **String column** - store raw JSON as a string, query with JSON functions
2. **Object('json') type** - experimental native JSON type with sub-column access
3. **Flattened columns** - extract fields at insert time into typed columns
4. **Tuple/Array columns** - structured nested data

Each has different trade-offs in flexibility, performance, and ease of use.

## Approach 1: JSON Stored as String

The simplest approach stores JSON as a plain `String` column and uses `JSONExtract*` functions:

```sql
CREATE TABLE events_json_string (
    id UInt64,
    ts DateTime,
    payload String
) ENGINE = MergeTree()
ORDER BY (ts, id);

INSERT INTO events_json_string VALUES
(1, now(), '{"user_id": 42, "action": "login", "ip": "192.168.1.1"}'),
(2, now(), '{"user_id": 43, "action": "purchase", "amount": 99.99, "items": ["hat", "shirt"]}'),
(3, now(), '{"user_id": 42, "action": "logout", "session_ms": 12500}');
```

```sql
-- Extract fields using JSONExtract functions
SELECT
    id,
    JSONExtractUInt(payload, 'user_id') AS user_id,
    JSONExtractString(payload, 'action') AS action,
    JSONExtractFloat(payload, 'amount') AS amount
FROM events_json_string;

-- Filter on JSON field
SELECT * FROM events_json_string
WHERE JSONExtractString(payload, 'action') = 'purchase';

-- Extract nested field
SELECT JSONExtractString(payload, 'address', 'city') FROM events_json_string;
```

## JSONExtract Functions Reference

```sql
-- Type-specific extractors
SELECT
    JSONExtractString('{"name":"Alice"}', 'name'),     -- String
    JSONExtractUInt('{"count":42}', 'count'),           -- UInt64
    JSONExtractInt('{"score":-5}', 'score'),            -- Int64
    JSONExtractFloat('{"price":9.99}', 'price'),        -- Float64
    JSONExtractBool('{"active":true}', 'active'),       -- UInt8 (0 or 1)
    JSONExtractArrayRaw('{"tags":["a","b"]}', 'tags'),  -- String (raw JSON array)
    JSONExtractRaw('{"x":{"y":1}}', 'x');               -- String (raw JSON value)

-- Dynamic type extraction
SELECT JSONExtract('{"val":42}', 'val', 'UInt32');

-- Check if key exists
SELECT JSONHas('{"name":"Bob"}', 'name');     -- 1
SELECT JSONHas('{"name":"Bob"}', 'missing'); -- 0

-- Get JSON keys
SELECT JSONExtractKeys('{"a":1,"b":2,"c":3}');  -- ['a','b','c']
```

## Approach 2: Native JSON Type (Object)

For better performance on frequently queried JSON fields:

```sql
SET allow_experimental_object_type = 1;

CREATE TABLE events_native_json (
    id UInt64,
    ts DateTime,
    payload Object('json')
) ENGINE = MergeTree()
ORDER BY (ts, id);

INSERT INTO events_native_json FORMAT JSONEachRow
{"id": 1, "ts": "2024-01-01 00:00:00", "payload": {"user_id": 42, "action": "login"}}
{"id": 2, "ts": "2024-01-01 00:01:00", "payload": {"user_id": 43, "action": "purchase", "amount": 99.99}}

-- Sub-column access with dot notation
SELECT payload.user_id, payload.action, payload.amount
FROM events_native_json;
```

## Approach 3: Flattened Columns with JSONExtract at Insert

Using a MATERIALIZED VIEW to extract fields:

```sql
-- Raw intake table
CREATE TABLE events_raw (
    raw_json String,
    received_at DateTime DEFAULT now()
) ENGINE = MergeTree()
ORDER BY received_at;

-- Processed table with typed columns
CREATE TABLE events_flat (
    user_id UInt64,
    action LowCardinality(String),
    amount Nullable(Float64),
    ip_address Nullable(String),
    session_ms Nullable(UInt64),
    received_at DateTime
) ENGINE = MergeTree()
ORDER BY (action, received_at);

-- Materialized view transforms raw -> flat
CREATE MATERIALIZED VIEW events_mv
TO events_flat AS
SELECT
    JSONExtractUInt(raw_json, 'user_id') AS user_id,
    JSONExtractString(raw_json, 'action') AS action,
    JSONExtractFloat(raw_json, 'amount') AS amount,
    JSONExtractString(raw_json, 'ip') AS ip_address,
    JSONExtractUInt(raw_json, 'session_ms') AS session_ms,
    received_at
FROM events_raw;

-- Insert to raw, flat table auto-populates
INSERT INTO events_raw (raw_json) VALUES
('{"user_id": 42, "action": "login", "ip": "10.0.0.1"}');
```

## Working with JSON Arrays

```sql
-- Extract array from JSON
SELECT
    id,
    JSONExtractArrayRaw(payload, 'items') AS items_raw,
    -- Use arrayMap for element extraction
    arrayMap(x -> trim('"' FROM x),
        JSONExtractArrayRaw(payload, 'items')
    ) AS items
FROM events_json_string
WHERE JSONHas(payload, 'items');

-- Explode JSON array into rows
SELECT
    id,
    arrayJoin(JSONExtract(payload, 'items', 'Array(String)')) AS item
FROM events_json_string
WHERE JSONHas(payload, 'items');
```

## Performance Best Practices

```sql
-- Index JSON fields by materializing them
CREATE TABLE events_optimized (
    id UInt64,
    ts DateTime,
    payload String,
    -- Materialized columns for common filter fields
    user_id UInt64 MATERIALIZED JSONExtractUInt(payload, 'user_id'),
    action LowCardinality(String) MATERIALIZED JSONExtractString(payload, 'action'),
    amount Float64 MATERIALIZED JSONExtractFloat(payload, 'amount')
) ENGINE = MergeTree()
ORDER BY (action, ts, user_id);

-- Now this query uses the materialized index instead of parsing JSON
SELECT count() FROM events_optimized
WHERE action = 'purchase' AND ts > now() - INTERVAL 7 DAY;
```

## Validating JSON

```sql
-- Check if a string is valid JSON
SELECT
    id,
    isValidJSON(payload) AS valid
FROM events_json_string;

-- Filter out invalid JSON rows
SELECT id, payload
FROM events_json_string
WHERE isValidJSON(payload) = 1;
```

## Summary

ClickHouse offers multiple strategies for JSON data: raw string storage with `JSONExtract*` functions provides maximum flexibility; the native `Object('json')` type offers sub-column access and better performance; and materialized view flattening gives the best query speed for known schemas. For high-volume analytics, materialize frequently filtered JSON fields as typed columns to avoid repeated JSON parsing at query time.
