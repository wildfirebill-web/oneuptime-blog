# How to Use Lightweight Deletes in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Lightweight Delete, GDPR, Data Management, MergeTree

Description: Learn how to use ClickHouse lightweight deletes for fast row removal without rewriting data parts, and when to choose them over ALTER TABLE DELETE.

---

ClickHouse introduced lightweight deletes as a faster alternative to mutations for removing rows. Traditional `ALTER TABLE DELETE` rewrites entire data parts, which is slow and resource-intensive. Lightweight deletes mark rows as deleted immediately and physically remove them during the next merge.

## How Lightweight Deletes Work

When you run `DELETE FROM`, ClickHouse writes a special deletion mask file alongside the data part. Queries immediately filter out masked rows. The actual data is physically removed during the background merge process.

## Basic Syntax

```sql
-- Enable lightweight deletes (required in older versions)
SET allow_experimental_lightweight_delete = 1;

-- Delete specific rows
DELETE FROM events WHERE user_id = 42;

-- Delete with date range
DELETE FROM events
WHERE event_time < '2023-01-01';

-- Delete by multiple conditions
DELETE FROM logs
WHERE level = 'debug'
  AND created_at < now() - INTERVAL 30 DAY;
```

## Verifying the Delete

After a lightweight delete, rows are immediately invisible to queries:

```sql
-- This returns 0 right after the DELETE
SELECT count() FROM events WHERE user_id = 42;

-- Check deletion mask status
SELECT table, rows, bytes_on_disk
FROM system.parts
WHERE table = 'events' AND active = 1;
```

## GDPR Right to Erasure

Lightweight deletes are practical for GDPR compliance where you need to remove a user's data quickly:

```sql
-- Remove all data for a user (GDPR erasure request)
DELETE FROM user_events WHERE user_id = :user_id;
DELETE FROM user_profiles WHERE user_id = :user_id;
DELETE FROM purchase_history WHERE user_id = :user_id;
```

For immediate physical removal (before the next merge), force a merge:

```sql
OPTIMIZE TABLE user_events FINAL;
```

## Lightweight Delete vs. ALTER TABLE DELETE

```sql
-- ALTER TABLE DELETE: rewrites data parts (slow, resource-heavy)
ALTER TABLE events DELETE WHERE user_id = 42;

-- Lightweight DELETE: marks rows immediately (fast, low overhead)
DELETE FROM events WHERE user_id = 42;
```

Choose lightweight deletes when:
- You need rows to disappear immediately from queries
- You delete rows frequently (e.g., expired data, GDPR)
- You want to avoid heavy I/O from part rewrites

Choose `ALTER TABLE DELETE` when:
- You need guaranteed physical removal before the next merge
- You are running on an older ClickHouse version without lightweight delete support

## Limitations

- Lightweight deletes are not supported on read-only replicas.
- They work on `MergeTree` family tables only.
- `DELETE FROM` without a WHERE clause deletes all rows but does not truncate the table; use `TRUNCATE TABLE` instead.
- On tables with projections, lightweight deletes rewrite projections immediately.

## Summary

Lightweight deletes in ClickHouse provide fast, low-overhead row removal by writing deletion masks rather than rewriting data parts. They are ideal for GDPR compliance, data expiry workflows, and any scenario where rows need to be invisible immediately without the cost of a full mutation.
