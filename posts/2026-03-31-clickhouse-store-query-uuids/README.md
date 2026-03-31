# How to Efficiently Store and Query UUIDs in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, UUID, Schema Design, Performance, Storage

Description: Learn best practices for storing and querying UUIDs in ClickHouse, covering native UUID type, FixedString(16), index design, and common performance pitfalls.

---

UUIDs are ubiquitous as surrogate keys and correlation identifiers in modern data systems, but their random nature clashes with ClickHouse's ordered-storage model. Choosing the right column type, sort key position, and query pattern dramatically affects both storage efficiency and query speed. This guide covers the trade-offs between the native `UUID` type and `FixedString(16)`, index strategies, and practical query patterns for UUID-heavy tables.

## Column Type Comparison

```sql
-- Compare storage types for UUIDs
SELECT
    'UUID'            AS type,
    16                AS bytes,
    'Two UInt64'      AS internal_rep,
    'Yes'             AS sortable,
    'Yes'             AS type_safe

UNION ALL SELECT 'FixedString(16)', 16, 'Raw bytes',       'Yes', 'No (cast needed)'
UNION ALL SELECT 'String',          37, 'Var-len + hyphen','Yes', 'No'
UNION ALL SELECT 'UInt128',         16, 'Single UInt128',  'Yes', 'No (manual)';
```

The native `UUID` type is almost always the right choice: it is 16 bytes (same as `FixedString(16)`), type-safe, and avoids the 37-byte overhead of storing UUIDs as strings.

## Declaring a UUID Column

```sql
CREATE TABLE events (
    id          UUID DEFAULT generateUUIDv4(),
    event_time  DateTime,
    user_id     UUID,
    session_id  UUID,
    event_type  LowCardinality(String),
    payload     String
) ENGINE = MergeTree
PARTITION BY toYYYYMM(event_time)
ORDER BY (event_time, id);
```

## UUID as the Only Sort Key - Avoid This

```sql
-- BAD: random UUID as sole sort key causes poor index locality
CREATE TABLE events_bad (
    id         UUID DEFAULT generateUUIDv4(),
    event_time DateTime,
    event_type String
) ENGINE = MergeTree
ORDER BY id;  -- random order = poor range scan performance
```

Because UUIDv4 is random, sorting by UUID alone means every row lands in a different block, forcing ClickHouse to read many granules for any range query.

## Good Sort Key Patterns for UUID Tables

```sql
-- GOOD: time-first sort key; UUID is a tiebreaker, not the primary dimension
CREATE TABLE events_good (
    id          UUID DEFAULT generateUUIDv4(),
    event_time  DateTime,
    event_type  LowCardinality(String),
    payload     String
) ENGINE = MergeTree
PARTITION BY toYYYYMM(event_time)
ORDER BY (event_time, id);

-- GOOD: when point lookups by user_id are common
CREATE TABLE user_events (
    user_id    UUID,
    event_time DateTime,
    id         UUID DEFAULT generateUUIDv4(),
    event_type LowCardinality(String)
) ENGINE = MergeTree
PARTITION BY toYYYYMM(event_time)
ORDER BY (user_id, event_time, id);
```

## Point Lookup by UUID with skipIndexes

For direct UUID lookups on large tables where UUID is not the leading sort key, a bloom filter skip index can dramatically reduce I/O.

```sql
CREATE TABLE events_indexed (
    id          UUID DEFAULT generateUUIDv4(),
    event_time  DateTime,
    user_id     UUID,
    event_type  LowCardinality(String),
    INDEX idx_id id TYPE bloom_filter(0.01) GRANULARITY 4
) ENGINE = MergeTree
PARTITION BY toYYYYMM(event_time)
ORDER BY (event_time, id);

-- Point lookup benefits from the bloom filter
SELECT *
FROM events_indexed
WHERE id = '550e8400-e29b-41d4-a716-446655440000';
```

## Querying UUIDs Correctly

```sql
-- Use the UUID literal form - no quotes as String
SELECT *
FROM events
WHERE id = '550e8400-e29b-41d4-a716-446655440000'  -- ClickHouse auto-casts String literal
LIMIT 1;

-- Explicit cast (clearer intent)
SELECT *
FROM events
WHERE id = toUUID('550e8400-e29b-41d4-a716-446655440000')
LIMIT 1;
```

## Batch Lookup by UUID List

```sql
-- Fetch multiple events by a list of UUIDs
SELECT
    id,
    event_time,
    event_type
FROM events
WHERE id IN (
    '550e8400-e29b-41d4-a716-446655440000',
    'f47ac10b-58cc-4372-a567-0e02b2c3d479',
    '6ba7b810-9dad-11d1-80b4-00c04fd430c8'
);
```

## Aggregating by UUID Key

```sql
-- Count events per user (user_id is UUID)
SELECT
    user_id,
    count()                          AS event_count,
    countIf(event_type = 'purchase') AS purchases,
    min(event_time)                  AS first_seen,
    max(event_time)                  AS last_seen
FROM user_events
WHERE event_time >= today() - 30
GROUP BY user_id
ORDER BY event_count DESC
LIMIT 20;
```

## UUID Nil Value Checks

The nil UUID (`00000000-0000-0000-0000-000000000000`) is ClickHouse's zero value for the UUID type - equivalent to NULL in some patterns.

```sql
-- Identify rows with unassigned UUIDs
SELECT count() AS unassigned
FROM events
WHERE id = toUUID('00000000-0000-0000-0000-000000000000');
```

## Storage Size Comparison

```sql
-- Compare table size for UUID vs String storage
SELECT
    table,
    sum(data_compressed_bytes)   AS compressed_bytes,
    sum(data_uncompressed_bytes) AS uncompressed_bytes,
    round(sum(data_compressed_bytes) / 1e6, 2) AS compressed_mb
FROM system.parts
WHERE table IN ('events_uuid', 'events_string')
  AND active = 1
GROUP BY table;
```

UUID columns compress significantly better than String UUID columns because the 16-byte binary representation has more repetition than hyphenated hex strings.

## Summary

Always use the native `UUID` column type over `String` or `FixedString(16)` for UUID fields: it is equally compact at 16 bytes, type-safe, and compresses well. Avoid making UUID the leading sort key on analytical tables - place a time dimension first to maintain block locality. Add a bloom filter skip index on UUID columns used for point lookups that are not covered by the sort key. Use `toUUID()` for explicit casts and `generateUUIDv4()` as a `DEFAULT` expression for auto-assigned surrogate keys.
