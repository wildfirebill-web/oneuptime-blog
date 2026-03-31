# How to Use murmurHash2_32() and murmurHash2_64() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Hash Function, MurmurHash, Sharding, A/B Testing

Description: Learn how to use murmurHash2_32() and murmurHash2_64() in ClickHouse for fast non-cryptographic hashing, A/B test assignment, and consistent sharding.

---

MurmurHash2 is a fast, non-cryptographic hash function with good distribution properties. ClickHouse provides `murmurHash2_32()` which returns a UInt32, and `murmurHash2_64()` which returns a UInt64. Both accept one or more arguments of any type. They are particularly popular for consistent sharding and A/B test assignment due to their speed and uniform distribution.

## Basic Usage

```sql
-- 32-bit MurmurHash2
SELECT murmurHash2_32('hello world') AS mhash32;

-- 64-bit MurmurHash2
SELECT murmurHash2_64('hello world') AS mhash64;

-- Hash multiple arguments together
SELECT murmurHash2_64(user_id, session_id) AS composite_hash
FROM sessions
LIMIT 5;
```

## A/B Test Assignment

One of the most important use cases for `murmurHash2_64` is stable A/B test assignment. Users should consistently land in the same experiment group across sessions.

```sql
-- Assign users to A/B test groups (group 0 = control, group 1 = treatment)
SELECT
    user_id,
    murmurHash2_64(user_id, 'experiment_v1') % 2 AS ab_group,
    CASE murmurHash2_64(user_id, 'experiment_v1') % 2
        WHEN 0 THEN 'control'
        WHEN 1 THEN 'treatment'
    END AS group_name
FROM users
LIMIT 20;
```

Adding the experiment name as a second argument (seed) ensures that different experiments produce different group assignments for the same user.

## Multi-Variant Testing

For tests with more than two variants, use modulo with the number of variants.

```sql
-- Split users into 4 experiment groups
SELECT
    user_id,
    murmurHash2_64(user_id, 'pricing_test_q2') % 4 AS variant
FROM users
ORDER BY variant;

-- Count users in each variant to verify uniform distribution
SELECT
    murmurHash2_64(user_id, 'pricing_test_q2') % 4 AS variant,
    count() AS user_count
FROM users
GROUP BY variant
ORDER BY variant;
```

## Consistent Sharding

Use `murmurHash2_64` to distribute data across shards in a stable and repeatable way.

```sql
-- Assign rows to 8 shards
SELECT
    order_id,
    customer_id,
    murmurHash2_64(customer_id) % 8 AS shard_id
FROM orders
LIMIT 20;
```

## Hashing Multiple Columns

Both functions accept multiple arguments, hashing them together as a composite key.

```sql
-- Hash a combination of user, event type, and date
SELECT
    user_id,
    event_type,
    toDate(event_time) AS event_date,
    murmurHash2_64(user_id, event_type, toDate(event_time)) AS composite_hash
FROM events
LIMIT 10;
```

## Deterministic Sampling

```sql
-- Sample 20% of users deterministically using a fixed seed
SELECT
    user_id,
    email
FROM users
WHERE murmurHash2_64(user_id) % 5 = 0
LIMIT 100;
```

## Comparing murmurHash2_32 and murmurHash2_64

```sql
SELECT
    'test value'                          AS input,
    murmurHash2_32('test value')          AS mh2_32,
    murmurHash2_64('test value')          AS mh2_64;
```

Prefer `murmurHash2_64` for most use cases as it provides better collision resistance with 64 bits of output. Use `murmurHash2_32` when you need a compact unsigned 32-bit integer, for example when fitting into UInt32 column types.

## Materialized Hash Column

```sql
CREATE TABLE user_experiments
(
    user_id        UInt64,
    experiment_id  String,
    event_time     DateTime,
    variant        UInt8 MATERIALIZED murmurHash2_64(user_id, experiment_id) % 4
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_time)
ORDER BY (user_id, experiment_id, event_time);
```

## Summary

`murmurHash2_32()` and `murmurHash2_64()` are fast, well-distributed hash functions ideal for A/B test assignment and consistent sharding. Their key advantage is that they accept multiple arguments directly, making composite hashing convenient without manual concatenation. Use a seed argument (like an experiment name) to generate independent hash spaces for different experiments. Neither function is suitable for cryptographic use.
