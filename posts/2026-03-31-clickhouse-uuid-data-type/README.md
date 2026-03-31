# How to Use UUID Data Type in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, Data Type, UUID, Identifier

Description: Learn how to use the UUID data type in ClickHouse - including 16-byte storage, generateUUIDv4(), string conversion, and efficient querying by UUID.

---

UUIDs (Universally Unique Identifiers) are a common choice for primary keys and correlation IDs in distributed systems because they can be generated without a central authority. ClickHouse has a native UUID type that stores each value as 16 bytes - far more efficient than storing a UUID as a 36-character String. This post covers how to declare UUID columns, generate UUIDs, convert between string and UUID formats, and query efficiently by UUID.

## UUID Storage

The UUID type stores a 128-bit value in 16 bytes. By comparison, storing a UUID as a String requires 36 bytes for the canonical `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx` form. The native UUID type cuts storage by more than half and enables faster equality and range comparisons.

A nil UUID (all zeros) is the default value for UUID columns when no value is provided: `00000000-0000-0000-0000-000000000000`.

## Creating Tables with UUID Columns

```sql
CREATE TABLE sessions
(
    session_id   UUID,
    user_id      UInt64,
    trace_id     UUID,
    ip_address   String,
    started_at   DateTime,
    ended_at     Nullable(DateTime)
)
ENGINE = MergeTree()
ORDER BY session_id;
```

## Generating UUIDs

Use `generateUUIDv4()` to generate a random version 4 UUID at query or insert time.

```sql
-- Generate a UUID on the fly
SELECT generateUUIDv4() AS new_uuid;

-- Insert rows with generated UUIDs
INSERT INTO sessions
    (session_id, user_id, trace_id, ip_address, started_at) VALUES
(generateUUIDv4(), 1001, generateUUIDv4(), '10.0.0.1', now()),
(generateUUIDv4(), 1002, generateUUIDv4(), '10.0.0.2', now()),
(generateUUIDv4(), 1003, generateUUIDv4(), '10.0.0.3', now());
```

## Converting Between UUID and String

Use `toString()` to convert a UUID to its canonical string representation, and `toUUID()` to parse a UUID string into the native UUID type.

```sql
-- UUID to string
SELECT
    session_id,
    toString(session_id)   AS session_id_string,
    length(toString(session_id)) AS string_length   -- Always 36
FROM sessions
LIMIT 3;

-- String to UUID
SELECT toUUID('550e8400-e29b-41d4-a716-446655440000') AS parsed_uuid;
```

## Querying by UUID

When filtering by UUID, use the native UUID literal syntax rather than a string comparison to avoid implicit casting overhead.

```sql
-- Using a UUID literal for efficient lookup
SELECT *
FROM sessions
WHERE session_id = toUUID('550e8400-e29b-41d4-a716-446655440000');

-- Parameterized approach with string input
SELECT *
FROM sessions
WHERE toString(session_id) = '550e8400-e29b-41d4-a716-446655440000';
```

## Using UUID as a Primary Key

UUID is commonly used as a distributed-friendly primary key. When using it as the ordering key in MergeTree, be aware that random UUIDs do not cluster well - consider a composite key.

```sql
CREATE TABLE events
(
    event_id     UUID DEFAULT generateUUIDv4(),
    event_type   LowCardinality(String),
    user_id      UInt64,
    payload      String,
    created_at   DateTime DEFAULT now()
)
ENGINE = MergeTree()
ORDER BY (event_type, created_at, event_id);

INSERT INTO events (event_type, user_id, payload) VALUES
('click',  1001, '{"element":"btn-signup"}'),
('view',   1002, '{"page":"/pricing"}'),
('submit', 1001, '{"form":"contact"}');
```

## UUID in JOIN Operations

UUID columns work naturally in JOIN conditions.

```sql
CREATE TABLE session_metadata
(
    session_id   UUID,
    browser      String,
    os           String
)
ENGINE = MergeTree()
ORDER BY session_id;

SELECT
    s.session_id,
    s.user_id,
    s.ip_address,
    m.browser,
    m.os
FROM sessions AS s
LEFT JOIN session_metadata AS m ON s.session_id = m.session_id;
```

## Storing UUID as FixedString(16)

For maximum storage efficiency, advanced users sometimes store UUIDs as `FixedString(16)` using raw binary form. However, this requires manual encoding and decoding and loses the semantic clarity of the UUID type.

```sql
-- UUID type is preferred over FixedString(16) for clarity
-- But if you need binary storage and manual control:
SELECT
    generateUUIDv4() AS uuid_native,
    toFixedString(reinterpretAsFixedString(generateUUIDv4()), 16) AS uuid_binary;
```

## Summary

ClickHouse's native UUID type stores 128-bit identifiers in just 16 bytes, making it significantly more compact than storing UUIDs as strings. Use `generateUUIDv4()` to produce random UUIDs at insert time, `toString()` and `toUUID()` for conversions, and native UUID literals in WHERE clauses for efficient filtering. For tables where UUID is the primary key, pair it with a meaningful prefix column in the ORDER BY clause to avoid poor data locality from random UUID ordering.
