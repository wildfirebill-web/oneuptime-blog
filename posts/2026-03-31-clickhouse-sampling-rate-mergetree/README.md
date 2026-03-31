# How to Configure Sampling Rate in MergeTree Tables

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, MergeTree, Sampling, SAMPLE BY, Table Design

Description: Learn how to configure the SAMPLE BY clause in ClickHouse MergeTree tables to enable consistent sampling and approximate query acceleration.

---

For the `SAMPLE` clause to work on a ClickHouse query, the underlying MergeTree table must declare a `SAMPLE BY` expression. This post covers how to configure `SAMPLE BY` correctly during table design and how it affects query behavior.

## How SAMPLE BY Works

`SAMPLE BY` defines the column (or expression) used to assign each row to a hash shard. ClickHouse computes `sipHash64(sample_key) % (2^64)` and uses the result to divide rows into deterministic subsets. When you issue `SAMPLE 0.1`, ClickHouse reads the shard covering 10% of the hash range.

## Basic SAMPLE BY Definition

```sql
CREATE TABLE page_views
(
    user_id    UInt64,
    page       String,
    event_time DateTime,
    duration_ms UInt32
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(event_time)
ORDER BY (user_id, event_time)
SAMPLE BY user_id;
```

`user_id` is a good sampling key because it ensures all events for a given user are either all included or all excluded from the sample - preserving user-level metrics.

## SAMPLE BY Must Be in ORDER BY

The `SAMPLE BY` column must be part of the `ORDER BY` (primary key). This ensures sorted storage enables efficient sampling:

```sql
-- Correct: user_id is first in ORDER BY
ORDER BY (user_id, event_time)
SAMPLE BY user_id

-- Incorrect: event_time is not in ORDER BY
ORDER BY user_id
SAMPLE BY event_time  -- will fail
```

## Using intHash32 for Even Distribution

For evenly distributed sampling, wrap the key in a hash function:

```sql
ORDER BY (intHash32(user_id), user_id, event_time)
SAMPLE BY intHash32(user_id);
```

This improves sampling accuracy when user IDs are not uniformly distributed.

## Verifying SAMPLE BY Configuration

```sql
SELECT name, sampling_key
FROM system.tables
WHERE database = 'default' AND name = 'page_views';
```

## Running Sampled Queries

```sql
-- 10% sample of today's page views
SELECT
    page,
    count() * 10 AS approx_views
FROM page_views
SAMPLE 0.1
WHERE toDate(event_time) = today()
GROUP BY page
ORDER BY approx_views DESC
LIMIT 20;
```

## Consistency Across Joins

When joining two sampled tables, use the same sampling key and fraction so matching rows appear in both samples:

```sql
SELECT
    e.user_id,
    e.event_name,
    u.plan
FROM events AS e SAMPLE 0.1
JOIN users AS u SAMPLE 0.1 USING (user_id)
WHERE toDate(e.event_time) = today();
```

Both tables must use `user_id` as their `SAMPLE BY` for consistent co-sampling.

## Adding SAMPLE BY to Existing Tables

You cannot add `SAMPLE BY` to an existing MergeTree table without recreating it. The safest approach is to create a new table and use `INSERT INTO ... SELECT`:

```sql
CREATE TABLE page_views_new ... ENGINE = MergeTree ... SAMPLE BY user_id;
INSERT INTO page_views_new SELECT * FROM page_views;
RENAME TABLE page_views TO page_views_old, page_views_new TO page_views;
```

## Summary

Configuring `SAMPLE BY` during table creation is a one-time design decision that unlocks fast approximate queries on any MergeTree table. Choose a key that ensures natural cohesion (such as `user_id`) and wrap it in `intHash32` for even distribution. With `SAMPLE BY` in place, dashboards can use `SAMPLE 0.1` for 10x query speedups with ~1% statistical error.
