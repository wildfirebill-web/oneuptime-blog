# How to Implement UPSERT Pattern in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, UPSERT, ReplacingMergeTree, Insert, Deduplication, Data Pattern

Description: Learn how to implement UPSERT semantics in ClickHouse using ReplacingMergeTree and insert-based patterns since ClickHouse lacks native INSERT OR UPDATE.

---

ClickHouse does not support traditional `INSERT OR UPDATE` (UPSERT) syntax found in PostgreSQL or MySQL. Instead, ClickHouse is designed around append-only ingestion with deduplication handled at merge time. This guide shows you the practical patterns for implementing UPSERT semantics.

## Pattern 1 - ReplacingMergeTree

The most common UPSERT pattern uses `ReplacingMergeTree`, which keeps the latest version of a row with the same primary key:

```sql
CREATE TABLE user_profiles (
    user_id UInt64,
    name String,
    email String,
    updated_at DateTime
) ENGINE = ReplacingMergeTree(updated_at)
ORDER BY user_id;
```

To "upsert", just insert the new version:

```sql
INSERT INTO user_profiles VALUES
    (42, 'Alice', 'alice@example.com', now());

-- Update: insert a newer version
INSERT INTO user_profiles VALUES
    (42, 'Alice Smith', 'alice@example.com', now());
```

During merges, ClickHouse keeps only the row with the highest `updated_at` value.

## Reading Deduplicated Data

Because deduplication happens at merge time, you must use `FINAL` to get consistent results before a merge occurs:

```sql
SELECT * FROM user_profiles FINAL WHERE user_id = 42;
```

Or use a subquery approach for better performance at scale:

```sql
SELECT *
FROM (
    SELECT *, row_number() OVER (PARTITION BY user_id ORDER BY updated_at DESC) AS rn
    FROM user_profiles
)
WHERE rn = 1;
```

## Pattern 2 - CollapsingMergeTree

For write-heavy UPSERT workloads, `CollapsingMergeTree` lets you explicitly cancel old rows:

```sql
CREATE TABLE order_status (
    order_id UInt64,
    status String,
    sign Int8
) ENGINE = CollapsingMergeTree(sign)
ORDER BY order_id;

-- Insert original
INSERT INTO order_status VALUES (100, 'pending', 1);

-- Upsert: cancel old row, insert new
INSERT INTO order_status VALUES
    (100, 'pending', -1),
    (100, 'shipped', 1);
```

## Pattern 3 - Using INSERT with Dedup

For ClickHouse Keeper-backed replicated tables, enable `insert_deduplicate`:

```sql
SET insert_deduplicate = 1;
INSERT INTO events VALUES (1, 'click', now());
-- Re-inserting the same data is safe - it will be deduplicated
INSERT INTO events VALUES (1, 'click', now());
```

This prevents duplicate inserts but does not handle key-based replacement.

## Summary

ClickHouse UPSERT is best implemented via `ReplacingMergeTree` for key-based deduplication with version tracking, `CollapsingMergeTree` for explicit row cancellation, or insert deduplication for idempotent writes. Choose the pattern that fits your consistency requirements and query read patterns.
