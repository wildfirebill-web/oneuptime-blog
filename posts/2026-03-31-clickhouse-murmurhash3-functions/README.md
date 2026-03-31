# How to Use murmurHash3_32() and murmurHash3_128() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Hash Function, MurmurHash, Deduplication, Sampling

Description: Learn how to use murmurHash3_32() and murmurHash3_128() in ClickHouse for better-distributed hashing, sampling, and deduplication workflows.

---

MurmurHash3 is an improved version of MurmurHash2 with better avalanche behavior and distribution. ClickHouse provides `murmurHash3_32()` returning a UInt32 and `murmurHash3_128()` returning a FixedString(16) containing a 128-bit hash. Both accept multiple arguments. MurmurHash3 is preferred over MurmurHash2 for new applications due to its superior distribution properties.

## Basic Usage

```sql
-- 32-bit MurmurHash3
SELECT murmurHash3_32('hello world') AS mh3_32;

-- 128-bit MurmurHash3 (returns FixedString(16))
SELECT hex(murmurHash3_128('hello world')) AS mh3_128_hex;

-- Hash multiple columns
SELECT murmurHash3_32(user_id, event_type) AS composite_hash
FROM events
LIMIT 5;
```

## Hash-Based Sampling

`murmurHash3_32` is excellent for deterministic percentage-based sampling.

```sql
-- Sample exactly 10% of users
SELECT
    user_id,
    email,
    created_at
FROM users
WHERE murmurHash3_32(user_id) % 10 = 0;

-- Sample 1% for a quick count estimate
SELECT
    count() * 100 AS estimated_total_events
FROM events
WHERE murmurHash3_32(user_id) % 100 = 0;
```

## Deduplication with murmurHash3_128

The 128-bit variant provides a very low collision probability, making it suitable for deduplication across large datasets.

```sql
-- Identify duplicate records by content hash
SELECT
    hex(murmurHash3_128(
        toString(user_id),
        toString(item_id),
        toString(toDate(event_time))
    )) AS record_hash,
    count() AS occurrences
FROM purchase_events
GROUP BY record_hash
HAVING occurrences > 1
ORDER BY occurrences DESC
LIMIT 10;
```

## Feature Hashing for Machine Learning

Feature hashing (also called the "hashing trick") maps categorical features to a fixed-size vector space. `murmurHash3_32` is commonly used for this purpose.

```sql
-- Map product categories to a 1024-bucket feature space
SELECT
    product_id,
    category,
    murmurHash3_32(category) % 1024 AS feature_bucket
FROM products
LIMIT 20;

-- Count features per bucket to check distribution
SELECT
    murmurHash3_32(category) % 1024 AS feature_bucket,
    count() AS feature_count
FROM products
GROUP BY feature_bucket
ORDER BY feature_count DESC
LIMIT 10;
```

## Comparing Distribution Quality

You can verify that `murmurHash3_32` distributes values evenly by checking the standard deviation of bucket counts.

```sql
-- Check distribution quality across 100 buckets
SELECT
    stddevPop(bucket_count)  AS stddev,
    avg(bucket_count)        AS avg_count,
    min(bucket_count)        AS min_count,
    max(bucket_count)        AS max_count
FROM (
    SELECT
        murmurHash3_32(user_id) % 100 AS bucket,
        count()                        AS bucket_count
    FROM users
    GROUP BY bucket
);
```

A good hash function produces a low standard deviation, indicating even distribution.

## Comparing murmurHash3_32 and murmurHash3_128

```sql
SELECT
    'test value'                             AS input,
    murmurHash3_32('test value')             AS mh3_32,
    hex(murmurHash3_128('test value'))       AS mh3_128_hex;
```

Use `murmurHash3_32` when you need integer arithmetic (modulo, bucketing). Use `murmurHash3_128` when you need a compact fingerprint with very low collision probability.

## Row Fingerprinting for Change Detection

```sql
-- Create row fingerprints to detect changes between snapshots
SELECT
    a.record_id,
    hex(murmurHash3_128(a.name, a.email, toString(a.status))) AS fingerprint_v1,
    hex(murmurHash3_128(b.name, b.email, toString(b.status))) AS fingerprint_v2,
    murmurHash3_128(a.name, a.email, toString(a.status)) !=
    murmurHash3_128(b.name, b.email, toString(b.status)) AS changed
FROM snapshot_v1 AS a
JOIN snapshot_v2 AS b ON a.record_id = b.record_id
WHERE changed = 1
LIMIT 20;
```

## Materialized Column for Fast Lookup

```sql
CREATE TABLE documents
(
    doc_id      UInt64,
    title       String,
    body        String,
    created_at  DateTime,
    body_hash   UInt32 MATERIALIZED murmurHash3_32(body)
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(created_at)
ORDER BY (doc_id, created_at);
```

## Summary

`murmurHash3_32()` and `murmurHash3_128()` offer improved distribution over MurmurHash2. Use `murmurHash3_32` for bucketing, sampling, and feature hashing in machine learning pipelines. Use `murmurHash3_128` for high-quality row fingerprinting and deduplication where a 32-bit hash would have too many collisions. Both accept multiple arguments natively, eliminating the need for manual string concatenation.
