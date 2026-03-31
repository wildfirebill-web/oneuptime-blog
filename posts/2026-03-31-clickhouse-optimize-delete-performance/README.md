# How to Optimize Delete Performance in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Delete Performance, Lightweight Delete, Mutation, Optimization, MergeTree

Description: Learn strategies to maximize delete performance in ClickHouse including partition pruning, lightweight deletes, and avoiding mutation bottlenecks.

---

Deletes in ClickHouse are fundamentally different from row-store databases. Because data is stored in columnar parts, deletions are most efficient when they align with the table's partitioning and sorting keys. Poor delete patterns can cause full part rewrites and degrade query performance.

## Strategy 1 - Delete by Partition

The fastest delete in ClickHouse is dropping an entire partition:

```sql
ALTER TABLE events DROP PARTITION '2024-01';
```

This is an O(1) operation - ClickHouse simply removes the partition directory. Design your partitioning key with eventual deletion in mind.

## Strategy 2 - Use Lightweight DELETE

For row-level deletes, lightweight `DELETE` is significantly faster than `ALTER TABLE ... DELETE` mutations because it only writes a small mask file:

```sql
DELETE FROM events WHERE user_id = 42;
```

Vs. the slower mutation approach:

```sql
-- Avoid for performance-sensitive cases
ALTER TABLE events DELETE WHERE user_id = 42;
```

## Strategy 3 - Filter on Primary Key Columns

Deletes that filter on the primary key (ORDER BY columns) enable part-level pruning, reducing the number of parts that need processing:

```sql
-- Efficient: event_date is the first ORDER BY column
DELETE FROM events WHERE event_date = '2024-03-15';

-- Less efficient: no primary key alignment
DELETE FROM events WHERE session_id = 'abc-xyz';
```

## Strategy 4 - Batch Deletes

Combine many small deletes into one operation to reduce overhead:

```sql
DELETE FROM users WHERE user_id IN (
    SELECT user_id FROM delete_queue WHERE processed = 0
);
```

## Strategy 5 - TTL Instead of Manual Deletes

For time-based retention, TTL eliminates the need for manual deletes entirely:

```sql
ALTER TABLE events
MODIFY TTL event_date + INTERVAL 90 DAY;
```

ClickHouse automatically removes expired rows during merges without any mutation overhead.

## Checking Mutation Efficiency

After a mutation completes, compare parts processed:

```sql
SELECT
    mutation_id,
    command,
    parts_to_do_names,
    is_done,
    latest_failed_part
FROM system.mutations
WHERE table = 'events'
ORDER BY create_time DESC
LIMIT 5;
```

## Summary

Optimize ClickHouse delete performance by aligning deletes with partition boundaries, using lightweight DELETE over mutations, filtering on primary key columns, batching operations, and replacing manual deletes with TTL policies where possible. The best delete is one that never needs to happen - design schemas with retention in mind.
