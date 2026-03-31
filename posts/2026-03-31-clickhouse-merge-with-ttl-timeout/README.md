# How to Use merge_with_ttl_timeout Setting in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, TTL, MergeTree, merge_with_ttl_timeout, Data Lifecycle, Settings

Description: Learn how the merge_with_ttl_timeout setting controls how frequently ClickHouse runs TTL-enforcing merges on MergeTree tables.

---

## What Is merge_with_ttl_timeout?

When you define a TTL rule on a MergeTree table, ClickHouse does not immediately delete or move expired data. Instead, TTL is enforced during background merge operations. The `merge_with_ttl_timeout` setting controls the minimum interval (in seconds) between merges that apply TTL rules.

The default value is 14400 seconds (4 hours).

## Why This Setting Exists

Applying TTL during every merge would be expensive. ClickHouse decouples regular compaction merges from TTL-enforcing merges using this timeout. TTL-enforcing merges are heavier because they:

- Evaluate expiry conditions row by row
- Physically remove or relocate expired data
- Rewrite affected parts

By spacing them out, ClickHouse reduces the I/O impact on a running system.

## Checking the Current Value

```sql
SELECT value
FROM system.merge_tree_settings
WHERE name = 'merge_with_ttl_timeout';
```

Or check the server-level setting:

```sql
SELECT name, value
FROM system.settings
WHERE name = 'merge_with_ttl_timeout';
```

## Setting merge_with_ttl_timeout Per Table

You can override the default per table at creation time:

```sql
CREATE TABLE events (
    event_time  DateTime,
    user_id     UInt64,
    message     String,
    TTL event_time + INTERVAL 30 DAY DELETE
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_time)
ORDER BY (event_time, user_id)
SETTINGS merge_with_ttl_timeout = 3600;  -- 1 hour
```

Or modify an existing table:

```sql
ALTER TABLE events
    MODIFY SETTING merge_with_ttl_timeout = 3600;
```

## Forcing TTL Application Immediately

If you need TTL to apply immediately without waiting for the next scheduled merge, use `OPTIMIZE`:

```sql
-- Force a TTL-aware merge on one partition
OPTIMIZE TABLE events PARTITION '2024-01' FINAL;

-- Force across all partitions (use carefully on large tables)
OPTIMIZE TABLE events FINAL;
```

`OPTIMIZE FINAL` forces all parts in a partition to merge into a single part and apply TTL.

## Balancing Freshness vs. I/O Cost

| Scenario | Recommended timeout |
|---|---|
| Dev or test environment | 300-600 seconds |
| Active OLAP workload | 3600-7200 seconds |
| Default (low churn) | 14400 seconds (4 hours) |
| Archive tables | 86400 seconds (24 hours) |

Reducing the timeout too aggressively on a high-write table will compete with regular merges and insert performance.

## Monitoring TTL Merge Activity

```sql
SELECT
    table,
    partition,
    merge_reason,
    event_time
FROM system.part_log
WHERE event_type = 'MergeParts'
  AND merge_reason = 'TTLMerge'
ORDER BY event_time DESC
LIMIT 20;
```

This shows when TTL merges last ran and which partitions were affected.

## Verifying Expired Data Is Removed

After a TTL merge completes, confirm data is gone:

```sql
SELECT count()
FROM events
WHERE event_time < now() - INTERVAL 30 DAY;
```

If rows remain, the TTL merge has not yet run for those parts. Trigger it with `OPTIMIZE`.

## Summary

`merge_with_ttl_timeout` controls how frequently ClickHouse applies TTL rules during background merges. The default 4-hour window balances TTL freshness against I/O overhead. Reduce it for time-sensitive data removal, increase it for archive tables where stale data costs more than latency. Use `OPTIMIZE TABLE FINAL` when you need immediate TTL enforcement.
