# How to Use farmHash64() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Hash Function, FarmHash, Partition Key, Sampling

Description: Learn how to use farmHash64() in ClickHouse for fast 64-bit Google FarmHash computation, unique ID generation, and deterministic sampling.

---

`farmHash64()` implements Google's FarmHash algorithm and returns a UInt64. FarmHash is designed to be fast on modern hardware and produces well-distributed values. Like `cityHash64()`, it accepts multiple arguments of any type. It is a good alternative to CityHash when you need a different hash space or want to compare distribution quality.

## Basic Usage

```sql
-- Hash a single string value
SELECT farmHash64('hello world') AS farm_hash;

-- Hash multiple columns together
SELECT
    user_id,
    page_url,
    farmHash64(user_id, page_url) AS composite_hash
FROM page_views
LIMIT 10;
```

## Generating Unique IDs

Use `farmHash64` to generate stable surrogate IDs from natural keys. The same combination of inputs always produces the same output.

```sql
-- Generate stable UInt64 IDs from composite natural keys
SELECT
    farmHash64(
        toString(order_id),
        toString(product_id),
        toString(warehouse_id)
    ) AS unique_line_id,
    order_id,
    product_id,
    warehouse_id
FROM order_lines
LIMIT 10;
```

## Partition Key Assignment

FarmHash is useful as a partition key in distributed setups where you want rows to be evenly distributed across shards.

```sql
-- Assign to one of 16 shards based on customer_id
SELECT
    customer_id,
    farmHash64(customer_id) % 16 AS shard_id
FROM customers
LIMIT 20;

-- Verify shard distribution is even
SELECT
    farmHash64(customer_id) % 16 AS shard_id,
    count()                       AS customers_in_shard
FROM customers
GROUP BY shard_id
ORDER BY shard_id;
```

## Deterministic Sampling

`farmHash64` supports stable percentage-based sampling.

```sql
-- Sample 5% of events deterministically
SELECT
    event_id,
    user_id,
    event_type,
    event_time
FROM events
WHERE farmHash64(event_id) % 20 = 0
LIMIT 1000;

-- Estimate total with 5% sample
SELECT
    event_type,
    count() * 20 AS estimated_count
FROM events
WHERE farmHash64(event_id) % 20 = 0
GROUP BY event_type
ORDER BY estimated_count DESC;
```

## Comparing farmHash64 with cityHash64

Both functions are fast 64-bit hashes. The main difference is the underlying algorithm, which produces different hash values for the same input.

```sql
-- Compare outputs for the same input
SELECT
    'comparison test'                        AS input,
    farmHash64('comparison test')            AS farm_hash,
    cityHash64('comparison test')            AS city_hash,
    farmHash64('comparison test') = cityHash64('comparison test') AS same_output;
```

The outputs will differ. Choosing between them often comes down to which algorithm is already in use in your existing infrastructure.

## Row Fingerprinting

```sql
-- Fingerprint rows for change detection
SELECT
    record_id,
    farmHash64(name, email, phone, address) AS row_fingerprint
FROM contacts
LIMIT 10;
```

## Using farmHash64 in a Table Definition

```sql
CREATE TABLE distributed_events
(
    event_id        UInt64,
    user_id         UInt64,
    event_type      String,
    event_time      DateTime,
    shard_key       UInt64 MATERIALIZED farmHash64(user_id)
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_time)
ORDER BY (user_id, event_time);
```

## Cross-Column Hash for Lookup Tables

When building a mapping from several columns to a single identifier, `farmHash64` provides a compact key.

```sql
-- Build a lookup from (source_id, entity_type) to a hash key
SELECT
    source_id,
    entity_type,
    farmHash64(source_id, entity_type) AS lookup_key
FROM entity_registry
LIMIT 20;
```

## Summary

`farmHash64()` is a fast, well-distributed 64-bit hash function from Google's FarmHash family. It accepts multiple arguments directly and is suitable for generating unique IDs, assigning partition keys, deterministic sampling, and row fingerprinting. It produces different values than `cityHash64` for the same inputs, so do not use both interchangeably in the same system without accounting for the difference.
