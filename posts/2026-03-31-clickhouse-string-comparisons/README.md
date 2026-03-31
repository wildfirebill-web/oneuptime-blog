# How to Optimize String Comparisons in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, String, Query Optimization, LowCardinality, Performance

Description: Optimize string comparison performance in ClickHouse using LowCardinality, FixedString, hashing, and proper WHERE clause patterns.

---

String comparisons are common but can be a significant source of query slowness in ClickHouse. The right data type and comparison method can make queries 10-100x faster.

## Use LowCardinality for Repeated Values

For columns with limited distinct values (status, country, event_type), `LowCardinality(String)` uses dictionary encoding and compares integers internally:

```sql
-- Slow: plain String for low-cardinality data
CREATE TABLE events (
  event_type String,
  user_id UInt64
) ENGINE = MergeTree() ORDER BY user_id;

-- Fast: LowCardinality reduces comparison to integer lookup
CREATE TABLE events_fast (
  event_type LowCardinality(String),
  user_id UInt64
) ENGINE = MergeTree() ORDER BY user_id;
```

Benchmark the difference:

```sql
SELECT count() FROM events WHERE event_type = 'purchase';       -- String
SELECT count() FROM events_fast WHERE event_type = 'purchase';  -- LowCardinality
```

## Use FixedString for Known-Length Strings

For fixed-length identifiers like ISO country codes or UUIDs stored as strings:

```sql
CREATE TABLE geo_events (
  country_code FixedString(2),  -- 'US', 'GB', 'DE'
  user_id UInt64,
  event_time DateTime
) ENGINE = MergeTree()
ORDER BY (country_code, event_time);
```

`FixedString` comparisons are faster than `String` because length is known at compile time.

## Avoid LIKE When Possible

`LIKE` with a leading wildcard disables index usage and requires scanning every row:

```sql
-- SLOW: leading wildcard, full scan
SELECT count() FROM logs WHERE message LIKE '%error%';

-- FAST: use hasToken for full-text token search
SELECT count() FROM logs WHERE hasToken(message, 'error');

-- FAST: use match() with regex anchored to start
SELECT count() FROM logs WHERE match(message, '^ERROR');
```

## Use Hashing for Exact Equality

For very long strings compared for exact equality, compare hashes instead:

```sql
-- Store hash alongside the string
ALTER TABLE urls ADD COLUMN url_hash UInt64 DEFAULT cityHash64(url);

-- Query using hash (fast) + string (verify)
SELECT * FROM urls
WHERE url_hash = cityHash64('https://example.com/very/long/path')
  AND url = 'https://example.com/very/long/path';
```

## Optimize IN Clause with Strings

For large IN lists of strings, use a dictionary or join:

```sql
-- SLOW for large lists
SELECT count() FROM events WHERE event_type IN ('purchase', 'refund', 'signup', ...);

-- FAST: use a dictionary
CREATE DICTIONARY event_type_allowlist (
  event_type String
) PRIMARY KEY event_type
SOURCE(CLICKHOUSE(query 'SELECT event_type FROM allowed_event_types'))
LAYOUT(HASHED())
LIFETIME(MIN 60 MAX 300);

SELECT count() FROM events
WHERE dictHas('event_type_allowlist', event_type);
```

## Summary

String comparisons in ClickHouse are optimized by using `LowCardinality(String)` for low-cardinality columns, `FixedString` for known-length identifiers, avoiding leading wildcards in `LIKE`, using `hasToken` for text search, and comparing hashes for exact equality on long strings. These changes reduce both CPU and I/O usage significantly.
