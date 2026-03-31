# How to Configure Merge Policies for MergeTree Tables

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, MergeTree, Merge Policy, Configuration, Performance, Part, SQL

Description: Learn how to configure ClickHouse MergeTree merge policies to control when and how data parts are merged, balancing write throughput against read performance.

---

ClickHouse MergeTree continuously merges small data parts into larger ones in the background. Merge policies control when merges happen, how large the resulting parts can be, and how aggressively the merge scheduler works. Tuning these settings prevents "Too many parts" errors and balances write throughput against query read amplification.

## How ClickHouse Selects Parts to Merge

ClickHouse uses a heuristic selector that considers:
1. Part age (older parts have higher merge priority).
2. Part size ratio (merge parts of similar size first to prevent large-part rewrite).
3. Partition boundary (merges never cross partition boundaries).

## Key Merge Policy Settings

### max_bytes_to_merge_at_max_space_in_pool

Maximum total size of parts that can be merged in one operation (when disk space is plentiful). Default: 150 GB.

```sql
ALTER TABLE events MODIFY SETTING max_bytes_to_merge_at_max_space_in_pool = 161061273600;
```

### max_bytes_to_merge_at_min_space_in_pool

When disk space is low, this lower limit applies. Default: 1 MB.

```sql
ALTER TABLE events MODIFY SETTING max_bytes_to_merge_at_min_space_in_pool = 1048576;
```

### merge_max_block_size

Maximum rows per block during a merge. Default: 8192.

```sql
ALTER TABLE events MODIFY SETTING merge_max_block_size = 8192;
```

Increase to 16384 for large tables to reduce merge overhead.

### number_of_free_entries_in_pool_to_lower_max_size_of_merge

When the merge thread pool has fewer free threads than this threshold, smaller merges are preferred. Default: 8.

```sql
ALTER TABLE events MODIFY SETTING number_of_free_entries_in_pool_to_lower_max_size_of_merge = 8;
```

### min_age_to_force_merge_seconds

Force-merges parts older than this age (in seconds). Default: 0 (disabled).

```sql
-- Force-merge parts older than 1 hour
ALTER TABLE events MODIFY SETTING min_age_to_force_merge_seconds = 3600;
```

Use for tables with predictable insert patterns to ensure old parts are always compacted.

### min_age_to_force_merge_on_partition_only

When enabled with `min_age_to_force_merge_seconds`, only merges if the entire partition qualifies:

```sql
ALTER TABLE events MODIFY SETTING min_age_to_force_merge_on_partition_only = 1;
```

## Monitoring Merge Activity

```sql
SELECT
    table,
    partition,
    result_part_name,
    elapsed,
    progress,
    num_parts,
    formatReadableSize(total_size_bytes_compressed) AS total_size
FROM system.merges
ORDER BY elapsed DESC;
```

## Checking Part Counts

High part counts indicate insufficient merge throughput:

```sql
SELECT
    table,
    partition,
    count() AS part_count,
    sum(rows) AS total_rows
FROM system.parts
WHERE active = 1
GROUP BY table, partition
HAVING part_count > 100
ORDER BY part_count DESC;
```

## Manually Triggering a Merge

For development or emergency compaction:

```sql
-- Merge all parts in a specific partition
OPTIMIZE TABLE events PARTITION '202603';

-- Force merge even if already optimized
OPTIMIZE TABLE events PARTITION '202603' FINAL;
```

## Background Pool Size

The number of concurrent merges is controlled by server-level settings in `config.xml`:

```text
<background_pool_size>16</background_pool_size>
<background_merges_mutations_concurrency_ratio>2</background_merges_mutations_concurrency_ratio>
```

Increase `background_pool_size` on write-heavy servers with many merge-pending tables.

## Summary

MergeTree merge policies balance write throughput against query read performance through background part merging. Key knobs are `max_bytes_to_merge_at_max_space_in_pool` (merge size ceiling), `min_age_to_force_merge_seconds` (compaction forcing), and `merge_max_block_size` (merge batch size). Monitor `system.merges` and `system.parts` part counts to detect merge lag and tune accordingly.
