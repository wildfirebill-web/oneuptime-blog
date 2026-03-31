# How to Use ALTER TABLE UPDATE in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, ALTER TABLE, Update, Mutation

Description: Learn how to update rows in ClickHouse using ALTER TABLE UPDATE, understand asynchronous mutations, and monitor them via system.mutations.

---

ClickHouse is designed for append-heavy analytical workloads, so row-level updates are handled differently than in transactional databases. Rather than an in-place row update, ClickHouse uses a mechanism called a mutation - a background operation that rewrites affected data parts. Understanding this model is essential before relying on `ALTER TABLE UPDATE` in production.

## Basic Syntax

The `ALTER TABLE ... UPDATE` statement follows this form:

```sql
ALTER TABLE table_name
UPDATE column1 = expr1, column2 = expr2
WHERE condition;
```

A simple example updating a status column:

```sql
ALTER TABLE events
UPDATE status = 'archived'
WHERE event_date < '2024-01-01';
```

Multiple columns can be updated in a single statement:

```sql
ALTER TABLE orders
UPDATE
    total_price = total_price * 1.1,
    updated_at = now()
WHERE region = 'EU' AND status = 'pending';
```

## How Mutations Work

When you issue `ALTER TABLE UPDATE`, ClickHouse does not modify rows in place. Instead, it schedules a mutation - a background job that reads the affected data parts, rewrites them with the updated values, and atomically replaces the old parts. This means:

- The mutation runs asynchronously after the statement returns.
- The original data remains visible until the rewrite completes.
- Mutations can be slow on large tables because they perform full part rewrites.

## Monitoring Mutations

Track the status of pending and completed mutations with `system.mutations`:

```sql
SELECT
    database,
    table,
    mutation_id,
    command,
    create_time,
    is_done,
    parts_to_do,
    latest_fail_reason
FROM system.mutations
WHERE table = 'events'
ORDER BY create_time DESC
LIMIT 10;
```

A mutation with `is_done = 0` and `parts_to_do > 0` is still running. If `latest_fail_reason` is non-empty, the mutation encountered an error.

To wait for a mutation to complete programmatically, poll until `is_done = 1`:

```sql
SELECT is_done
FROM system.mutations
WHERE table = 'events' AND mutation_id = 'mutation_1.txt';
```

## Killing a Mutation

If a mutation is taking too long or was issued by mistake, cancel it:

```sql
KILL MUTATION WHERE database = 'default' AND table = 'events' AND mutation_id = 'mutation_1.txt';
```

## Deduplication Concerns

ClickHouse's ReplicatedMergeTree engine uses a mutation deduplication log. If the same mutation command is submitted twice with the same checksum, the second submission is silently ignored. This protects against accidental double-application of mutations in replicated clusters, but it means retrying a failed mutation may require using a different query hash or waiting for the deduplication window to expire.

```sql
-- Check deduplication log entries
SELECT *
FROM system.mutations
WHERE is_done = 1 AND table = 'events'
ORDER BY create_time DESC
LIMIT 5;
```

## UPDATE with Subquery Values

You cannot use a subquery directly in the SET clause of `ALTER TABLE UPDATE`. Instead, use deterministic expressions or computed values:

```sql
-- Valid: expression based on existing columns
ALTER TABLE sessions
UPDATE duration_ms = end_ts - start_ts
WHERE duration_ms = 0;

-- Valid: use a function
ALTER TABLE logs
UPDATE level = lower(level)
WHERE level != lower(level);
```

## Performance Considerations

- Mutations rewrite entire parts, even if only a small fraction of rows match the WHERE clause.
- Avoid frequent small mutations; batch updates into one statement when possible.
- On large tables, consider partitioning strategies so mutations only touch relevant partitions.
- Use `ALTER TABLE ... UPDATE` on `MergeTree` family engines only. It is not supported on other engine types.

## Summary

`ALTER TABLE UPDATE` in ClickHouse triggers an asynchronous mutation that rewrites data parts rather than updating rows in place. Monitor progress through `system.mutations` and cancel unwanted operations with `KILL MUTATION`. For best performance, batch updates and prefer partitioned tables so mutations operate on smaller data sets.
