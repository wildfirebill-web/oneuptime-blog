# How to Use halfMD5() Function in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Hash Function, MD5, Row ID, Sampling

Description: Learn how to use halfMD5() in ClickHouse to compute compact 64-bit row IDs and deterministic samples from MD5 hashes.

---

`halfMD5()` computes the MD5 hash of all its arguments and returns the first 8 bytes (64 bits) interpreted as a UInt64. It is essentially a fast way to get a compact numeric hash from multiple values using the MD5 algorithm internally. Because it accepts multiple arguments, it is convenient for generating composite row IDs without string concatenation.

## Basic Usage

```sql
-- Hash a single value and get a UInt64
SELECT halfMD5('hello world') AS half_md5;

-- Hash multiple columns together
SELECT
    user_id,
    session_id,
    halfMD5(user_id, session_id) AS composite_hash
FROM sessions
LIMIT 10;
```

## Compact Row ID Generation

`halfMD5` is useful when you need a stable 64-bit integer identifier derived from multiple columns without using a 128-bit output like full MD5.

```sql
-- Generate a compact row ID from content fields
SELECT
    halfMD5(
        toString(user_id),
        toString(product_id),
        toString(toDate(event_time))
    ) AS row_id,
    user_id,
    product_id,
    event_time
FROM purchase_events
LIMIT 10;
```

## Deterministic Sampling

Like other hash functions, `halfMD5` enables reproducible sampling when used with modulo.

```sql
-- Sample 10% of rows deterministically
SELECT
    user_id,
    event_type,
    event_time
FROM events
WHERE halfMD5(user_id, toDate(event_time)) % 10 = 0
LIMIT 1000;
```

## Compatibility with Legacy Systems Using MD5

If an existing system generates row IDs by taking the first 8 bytes of an MD5 hash, `halfMD5` lets you replicate the same IDs in ClickHouse.

```sql
-- Replicate legacy MD5-based row IDs
SELECT
    record_id,
    legacy_row_id,
    halfMD5(toString(record_id), source_system) AS ch_row_id,
    legacy_row_id = halfMD5(toString(record_id), source_system) AS matches
FROM legacy_records
LIMIT 20;
```

## Using halfMD5 for Bucketing

```sql
-- Distribute data across 64 buckets
SELECT
    item_id,
    halfMD5(item_id, category) % 64 AS bucket
FROM items
LIMIT 20;

-- Check bucket distribution
SELECT
    halfMD5(item_id, category) % 64 AS bucket,
    count() AS items_in_bucket
FROM items
GROUP BY bucket
ORDER BY bucket;
```

## Comparing halfMD5 to Full MD5

```sql
-- Show that halfMD5 is the first 8 bytes of MD5
SELECT
    'test input'                          AS input,
    halfMD5('test input')                 AS half_md5_uint64,
    hex(MD5('test input'))                AS full_md5_hex,
    -- The first 16 chars of the MD5 hex correspond to the first 8 bytes
    substring(hex(MD5('test input')), 1, 16) AS first_8_bytes_hex;
```

## When to Use halfMD5 vs Other Hash Functions

```sql
-- Comparison of hash outputs and types
SELECT
    'sample'                  AS input,
    halfMD5('sample')         AS half_md5,    -- UInt64 from MD5
    cityHash64('sample')      AS city64,       -- UInt64 from CityHash
    farmHash64('sample')      AS farm64,       -- UInt64 from FarmHash
    murmurHash2_64('sample')  AS mh2_64;       -- UInt64 from MurmurHash2
```

`halfMD5` produces a UInt64 from the MD5 algorithm. It is slower than `cityHash64` or `xxHash64` since MD5 is more computationally expensive. Use `halfMD5` primarily for compatibility with legacy systems that already use MD5-based IDs.

## Materialized halfMD5 Column

```sql
CREATE TABLE legacy_compatible_events
(
    source_id    String,
    event_type   String,
    event_time   DateTime,
    row_id       UInt64 MATERIALIZED halfMD5(source_id, event_type)
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_time)
ORDER BY (source_id, event_time);
```

## Summary

`halfMD5()` returns the first 8 bytes of the MD5 hash of its arguments as a UInt64. It accepts multiple arguments directly, making composite key hashing convenient. Its primary use case is compatibility with legacy systems that derive numeric IDs from MD5 hashes. For new applications without MD5 compatibility requirements, prefer `cityHash64()` or `farmHash64()` which are faster.
