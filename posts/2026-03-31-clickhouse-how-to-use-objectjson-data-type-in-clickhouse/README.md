# How to Use Object('json') Data Type in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, JSON, Object Type, Dynamic Schema, Semi-Structured Data

Description: Learn how to use the experimental Object('json') data type in ClickHouse to store and query semi-structured JSON data with dynamic column access.

---

## What Is the Object('json') Type

The `Object('json')` type (also written as `JSON`) is an experimental ClickHouse data type that stores JSON objects as structured sub-columns. Unlike storing JSON as a plain `String`, ClickHouse parses the JSON at insert time and stores each path as a dedicated column - enabling efficient filtering and aggregation on nested fields.

```sql
-- Enable the experimental feature
SET allow_experimental_object_type = 1;

CREATE TABLE events (
    id UInt64,
    ts DateTime,
    payload Object('json')  -- or JSON
) ENGINE = MergeTree()
ORDER BY (ts, id);
```

## Inserting JSON Data

```sql
-- Insert JSON strings - ClickHouse parses them automatically
INSERT INTO events FORMAT JSONEachRow
{"id": 1, "ts": "2024-01-01 00:00:00", "payload": {"event": "click", "user_id": 42, "button": "submit"}}
{"id": 2, "ts": "2024-01-01 00:01:00", "payload": {"event": "view", "user_id": 43, "page": "/home"}}
{"id": 3, "ts": "2024-01-01 00:02:00", "payload": {"event": "purchase", "user_id": 42, "amount": 99.99, "items": ["shirt", "hat"]}}

-- Insert via VALUES with JSON string
INSERT INTO events VALUES
(4, now(), '{"event": "logout", "user_id": 44, "session_duration": 300}');
```

## Accessing Sub-Columns

Sub-columns are accessed using dot notation:

```sql
-- Access top-level JSON fields
SELECT
    id,
    payload.event,
    payload.user_id
FROM events;

-- Access nested fields
SELECT payload.items FROM events WHERE payload.event = 'purchase';

-- Mix regular columns and JSON sub-columns
SELECT
    ts,
    payload.event AS event_type,
    payload.user_id AS uid,
    payload.amount AS purchase_amount
FROM events
WHERE payload.event IN ('purchase', 'click');
```

## Filtering on JSON Sub-Columns

```sql
-- Filter by a JSON field value
SELECT * FROM events
WHERE payload.user_id = 42;

-- Range filter on numeric JSON fields
SELECT id, payload.amount
FROM events
WHERE payload.amount > 50.0;

-- Using JSON fields in GROUP BY
SELECT
    payload.event AS event_type,
    count() AS event_count,
    uniq(payload.user_id) AS unique_users
FROM events
GROUP BY payload.event
ORDER BY event_count DESC;
```

## How Sub-Columns Are Stored

Each distinct JSON path becomes a typed sub-column. You can inspect the inferred schema:

```sql
-- Check what sub-columns were created
DESCRIBE TABLE events;

-- Or query the system table
SELECT
    name,
    type
FROM system.columns
WHERE table = 'events'
  AND database = currentDatabase()
ORDER BY name;
```

ClickHouse infers types from the first inserted data. Later inserts with different types for the same path cause type promotion or errors.

## Limitations and Caveats

```sql
-- The Object type is experimental - enable it explicitly
SET allow_experimental_object_type = 1;

-- Sub-columns cannot be used in ORDER BY or PRIMARY KEY
-- This will error:
-- CREATE TABLE bad (payload JSON) ENGINE = MergeTree() ORDER BY payload.id;

-- Arrays in JSON become Array(Nullable(T)) sub-columns
-- {"tags": ["a", "b", "c"]} creates payload.tags Array(Nullable(String))

-- Deeply nested and mixed-type paths can cause issues
-- {"x": 1} then {"x": "hello"} - type conflict
```

## Using toJSONString and JSONExtract with Object Columns

```sql
-- Convert Object column back to JSON string
SELECT toJSONString(payload) FROM events LIMIT 5;

-- JSONExtract still works if you need dynamic path access
SELECT JSONExtractString(toJSONString(payload), 'event') FROM events;

-- Alternative: use JSON* functions for dynamic access
SELECT
    JSONExtractUInt(toJSONString(payload), 'user_id') AS user_id,
    JSONExtractFloat(toJSONString(payload), 'amount') AS amount
FROM events
WHERE payload.event = 'purchase';
```

## Practical Example: Application Event Tracking

```sql
SET allow_experimental_object_type = 1;

CREATE TABLE app_events (
    event_id UInt64,
    received_at DateTime DEFAULT now(),
    source LowCardinality(String),
    properties Object('json')
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(received_at)
ORDER BY (source, received_at, event_id);

-- Insert events with varying shapes
INSERT INTO app_events (event_id, source, properties) FORMAT JSONEachRow
{"event_id": 1, "source": "web", "properties": {"page": "/checkout", "user": 101, "cart_value": 250.00}}
{"event_id": 2, "source": "mobile", "properties": {"screen": "HomeScreen", "user": 102, "app_version": "2.1.0"}}
{"event_id": 3, "source": "web", "properties": {"page": "/products", "user": 101, "referrer": "google"}}

-- Analyze by source
SELECT
    source,
    count() AS events,
    uniq(properties.user) AS unique_users
FROM app_events
GROUP BY source;
```

## Migration Path and Future

The `Object('json')` type is the predecessor to the new `JSON` type in ClickHouse 24+. In newer versions, the JSON type is more stable and performant:

```sql
-- ClickHouse 24+ new JSON type (more stable)
SET enable_json_type = 1;

CREATE TABLE new_events (
    id UInt64,
    data JSON
) ENGINE = MergeTree()
ORDER BY id;
```

## Summary

The `Object('json')` data type in ClickHouse allows semi-structured JSON data to be stored with automatic schema inference and efficient sub-column access via dot notation. Each JSON path becomes a stored sub-column, enabling fast filtering and aggregation on nested fields without preprocessing. While still experimental in older versions, it is a powerful option for event tracking, log analytics, and other use cases where schema varies per record.
