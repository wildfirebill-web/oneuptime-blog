# How to Fix 'Mutation was killed' in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Mutation, Error, DELETE, UPDATE, Troubleshooting

Description: Fix 'Mutation was killed' errors in ClickHouse by identifying the cause, re-running mutations, and configuring mutation settings properly.

---

ClickHouse mutations (ALTER TABLE ... UPDATE/DELETE) are background operations. When a mutation is killed, you see: `DB::Exception: Mutation was killed`. This can happen due to server restart, explicit cancellation, memory exhaustion, or timeout.

## Check Mutation Status

```sql
SELECT
    database,
    table,
    mutation_id,
    command,
    create_time,
    is_done,
    latest_failed_part,
    latest_fail_reason,
    parts_to_do
FROM system.mutations
WHERE is_done = 0
ORDER BY create_time DESC;
```

The `latest_fail_reason` column shows why the mutation was killed.

## Identify If Mutation Was Explicitly Cancelled

```sql
SELECT * FROM system.mutations
WHERE latest_fail_reason LIKE '%killed%'
   OR latest_fail_reason LIKE '%cancelled%';
```

## Re-Run the Mutation

If the mutation was killed due to a server restart, simply re-submit it:

```sql
ALTER TABLE my_table DELETE WHERE event_date < '2024-01-01';
ALTER TABLE my_table UPDATE status = 'archived' WHERE event_date < '2024-01-01';
```

## Kill a Stuck Mutation Explicitly

If a mutation is consuming too many resources and you want to stop it:

```sql
KILL MUTATION WHERE database = 'my_database' AND table = 'my_table';
```

## Increase Mutation Memory Limit

Large mutations can be killed by memory limits:

```xml
<!-- config.xml -->
<mutations_memory_usage_soft_limit>0</mutations_memory_usage_soft_limit>
<background_pool_size>8</background_pool_size>
```

## Use Lightweight Deletes Instead

For ClickHouse 23.3+, lightweight deletes avoid mutation overhead:

```sql
-- Instead of ALTER TABLE DELETE (mutation):
DELETE FROM my_table WHERE event_date < '2024-01-01';
```

Lightweight deletes mark rows as deleted without rewriting parts immediately, avoiding the mutation killed problem.

## Partition-Based Alternative

For large deletes, drop partitions instead of running a mutation:

```sql
-- Check partition names
SELECT DISTINCT partition FROM system.parts
WHERE table = 'my_table' AND database = 'my_database';

-- Drop the partition (instant, no mutation needed)
ALTER TABLE my_table DROP PARTITION '2024-01';
```

## Monitor Mutation Progress

```sql
SELECT
    mutation_id,
    command,
    parts_to_do,
    parts_to_do_names
FROM system.mutations
WHERE table = 'my_table' AND is_done = 0;
```

## Summary

"Mutation was killed" in ClickHouse means a background ALTER UPDATE/DELETE was terminated. Check `system.mutations` for the failure reason, then re-run the mutation or use lightweight deletes (ClickHouse 23.3+). For large data removals, partition drops are the most efficient approach and avoid mutations entirely.
