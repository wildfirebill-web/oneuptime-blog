# How to Use ALTER TABLE DELETE in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, ALTER TABLE, Delete, Mutation

Description: Learn how to delete rows in ClickHouse using ALTER TABLE DELETE and lightweight DELETE, monitor mutations, and understand the performance trade-offs.

---

Deleting rows in ClickHouse requires understanding two distinct mechanisms: the traditional `ALTER TABLE DELETE` mutation and the newer lightweight `DELETE` statement. Both achieve row removal but differ significantly in how they operate, their performance characteristics, and their use cases. Choosing the right approach depends on your engine, data volume, and latency requirements.

## ALTER TABLE DELETE Syntax

The mutation-based delete rewrites entire data parts to remove matching rows:

```sql
ALTER TABLE table_name
DELETE WHERE condition;
```

Example - removing old event records:

```sql
ALTER TABLE events
DELETE WHERE event_date < '2023-01-01';
```

Delete rows matching multiple conditions:

```sql
ALTER TABLE user_activity
DELETE WHERE user_id = 42 AND action = 'spam';
```

## How the Mutation Executes

Like `ALTER TABLE UPDATE`, the DELETE mutation runs asynchronously. The statement schedules a background job that:

1. Reads each affected data part.
2. Filters out rows matching the WHERE clause.
3. Writes a new part without the deleted rows.
4. Atomically swaps the old part for the new one.

The data remains visible to queries until the part swap completes.

## Monitoring Mutations in system.mutations

```sql
SELECT
    database,
    table,
    mutation_id,
    command,
    create_time,
    is_done,
    parts_to_do,
    parts_to_do_names,
    latest_fail_reason
FROM system.mutations
WHERE table = 'events'
ORDER BY create_time DESC;
```

Check how many parts still need to be processed:

```sql
SELECT
    mutation_id,
    parts_to_do,
    is_done
FROM system.mutations
WHERE table = 'events' AND is_done = 0;
```

## Lightweight DELETE (ClickHouse 22.8+)

Lightweight `DELETE` marks rows as deleted without immediately rewriting parts. This makes it much faster for interactive workloads:

```sql
DELETE FROM events WHERE event_date < '2023-01-01';
```

Internally, ClickHouse writes a small deletion mask. Rows marked deleted are filtered at query time until the next merge removes them from the physical data. This approach is:

- Faster to acknowledge (no immediate part rewrite).
- Suitable for GDPR-style individual row deletions.
- Only available on `MergeTree` family tables with `allow_experimental_lightweight_delete = 1` on older versions.

```sql
-- Enable if not already set at session level
SET allow_experimental_lightweight_delete = 1;

DELETE FROM users WHERE user_id = 12345;
```

## Comparing Mutation DELETE vs Lightweight DELETE

```sql
-- Mutation DELETE: rewrites parts, fully removes data
ALTER TABLE events DELETE WHERE user_id = 99;

-- Lightweight DELETE: marks rows, deferred physical removal
DELETE FROM events WHERE user_id = 99;
```

| Aspect | ALTER TABLE DELETE | Lightweight DELETE |
|---|---|---|
| Execution | Asynchronous mutation | Synchronous mark |
| Speed | Slow on large data | Fast acknowledgment |
| Physical removal | Immediate (after mutation) | Deferred (on merge) |
| Monitoring | system.mutations | system.parts |

## Killing a Delete Mutation

```sql
KILL MUTATION
WHERE database = 'default'
  AND table = 'events'
  AND mutation_id = 'mutation_2.txt';
```

## Performance Considerations

- Mutations rewrite entire parts regardless of how many rows match the filter.
- On large tables, partition-level deletes are much faster because they operate on fewer parts.
- Prefer lightweight DELETE for frequent or selective row removals.
- Use `ALTER TABLE DROP PARTITION` when deleting an entire partition - this is instantaneous compared to a mutation.

```sql
-- Fastest way to remove a full partition
ALTER TABLE events DROP PARTITION '2023-01';
```

## Summary

`ALTER TABLE DELETE` schedules an asynchronous mutation that physically rewrites parts to remove rows, while lightweight `DELETE` marks rows instantly and defers physical cleanup to the next merge. Use mutations for bulk historical cleanup, lightweight DELETE for targeted row removal, and `DROP PARTITION` for the fastest full-partition purges. Always monitor long-running mutations via `system.mutations`.
