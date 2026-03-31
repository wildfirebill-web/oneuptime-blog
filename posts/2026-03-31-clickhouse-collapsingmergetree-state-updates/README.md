# How to Track State Updates with CollapsingMergeTree in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, CollapsingMergeTree, State Updates, Deduplication, OLAP

Description: Use ClickHouse CollapsingMergeTree to efficiently track mutable state by canceling old rows and inserting new ones without UPDATE operations.

---

ClickHouse is an append-only database, but many real-world workloads require mutable state - think user profiles, order statuses, or inventory counts. The `CollapsingMergeTree` engine solves this by using a sign column: insert a row with `sign = 1` to add state and `sign = -1` to cancel (collapse) it. Background merges physically remove the paired rows.

## Create a CollapsingMergeTree Table

```sql
CREATE TABLE user_sessions
(
    session_id  UInt64,
    user_id     UInt64,
    status      LowCardinality(String),
    page_views  UInt32,
    duration_s  UInt32,
    updated_at  DateTime,
    sign        Int8
)
ENGINE = CollapsingMergeTree(sign)
ORDER BY (session_id);
```

## Insert Initial State

```sql
INSERT INTO user_sessions VALUES
(1001, 42, 'active', 5, 120, now(), 1);
```

## Update State: Cancel Old Row and Insert New Row

```sql
-- Cancel the old row
INSERT INTO user_sessions VALUES
(1001, 42, 'active', 5, 120, '2026-03-31 10:00:00', -1);

-- Insert updated row
INSERT INTO user_sessions VALUES
(1001, 42, 'completed', 12, 540, now(), 1);
```

## Query Current State (Before Merge Happens)

Because merges happen asynchronously, you must account for sign in queries:

```sql
SELECT
    session_id,
    user_id,
    argMaxIf(status, updated_at, sign = 1)     AS status,
    sumIf(page_views, sign = 1)
        - sumIf(page_views, sign = -1)          AS page_views,
    sumIf(duration_s, sign = 1)
        - sumIf(duration_s, sign = -1)          AS duration_s
FROM user_sessions
GROUP BY session_id, user_id
HAVING sum(sign) > 0;
```

## Simpler Query Using FINAL

The `FINAL` modifier applies collapsing at query time, eliminating the sign arithmetic - at the cost of slower reads on large tables:

```sql
SELECT session_id, user_id, status, page_views, duration_s
FROM user_sessions FINAL
WHERE status = 'active';
```

## Verify Collapse After Optimize

```sql
OPTIMIZE TABLE user_sessions FINAL;

SELECT * FROM user_sessions WHERE session_id = 1001;
```

After the optimize, only the latest `sign = 1` row should remain.

## Batch State Updates

```sql
-- Cancel multiple old rows
INSERT INTO user_sessions
SELECT session_id, user_id, status, page_views, duration_s, updated_at, -1
FROM user_sessions FINAL
WHERE user_id = 42 AND status = 'active';

-- Insert updated rows
INSERT INTO user_sessions
SELECT session_id, user_id, 'expired', page_views, duration_s, now(), 1
FROM user_sessions FINAL
WHERE user_id = 42 AND status = 'active';
```

## Summary

`CollapsingMergeTree` enables efficient state tracking in ClickHouse without costly mutations. The pattern - cancel with `sign = -1`, reinsert with `sign = 1` - fits high-throughput pipelines where rows flow in from Kafka or batch jobs. Use `FINAL` for convenience during development and sign-aware aggregations in production for best performance.
