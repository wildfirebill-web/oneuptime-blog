# How to Use system.merges Table in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, system.merges, Background Merge, MergeTree, Monitoring, Performance

Description: Query system.merges to observe active background merge operations, track progress, and diagnose merge bottlenecks in ClickHouse MergeTree tables.

---

ClickHouse uses background merge processes to combine smaller data parts into larger ones, which is essential for query performance. The `system.merges` table provides a live view of all currently running merge operations, making it the primary tool for diagnosing merge backlog and performance issues.

## What is system.merges?

`system.merges` is a system table that shows currently active background merges for MergeTree family tables. Each row represents one ongoing merge, including its progress, estimated time remaining, and which parts are being merged.

Key columns include:
- `database`, `table` - target table
- `result_part_name` - name of the new part being created
- `elapsed` - seconds since the merge started
- `progress` - fraction completed (0.0 to 1.0)
- `total_size_bytes_compressed` - total compressed bytes to merge
- `bytes_read_uncompressed` - bytes already processed
- `rows_read` - rows already merged
- `merge_type` - type of merge (REGULAR, TTL, etc.)

## Viewing Active Merges

```sql
SELECT
    database,
    table,
    result_part_name,
    round(elapsed, 1)    AS elapsed_sec,
    round(progress, 3)   AS progress,
    formatReadableSize(total_size_bytes_compressed)  AS total_size,
    formatReadableSize(memory_usage)                 AS memory_usage,
    merge_type
FROM system.merges
ORDER BY elapsed DESC;
```

## Monitoring Merge Progress

Check if a specific table has long-running merges:

```sql
SELECT
    result_part_name,
    round(elapsed)                                     AS elapsed_sec,
    round(progress * 100, 1)                           AS pct_done,
    round(elapsed / nullif(progress, 0) - elapsed, 0) AS eta_sec
FROM system.merges
WHERE table = 'events'
ORDER BY elapsed DESC;
```

## Detecting Merge Backlog

If the number of active parts keeps growing despite merges running, check the queue depth:

```sql
-- Active parts per table
SELECT
    database,
    table,
    count() AS active_parts
FROM system.parts
WHERE active = 1
GROUP BY database, table
ORDER BY active_parts DESC
LIMIT 20;

-- Active merges count
SELECT count() AS running_merges
FROM system.merges;
```

When `active_parts` in a partition exceeds 300+, ClickHouse throttles inserts. Check `system.merges` to confirm merges are progressing.

## Merge Types

| merge_type | Description |
|------------|-------------|
| REGULAR | Standard part merge |
| TTL_DELETE | Expiring rows by TTL |
| TTL_RECOMPRESS | Recompressing old parts |

```sql
SELECT merge_type, count() AS count
FROM system.merges
GROUP BY merge_type;
```

## Stopping a Specific Table's Merges

If a merge is consuming too many resources:

```sql
SYSTEM STOP MERGES mydb.events;
-- investigate / take action
SYSTEM START MERGES mydb.events;
```

## Summary

`system.merges` provides real-time visibility into ClickHouse's background merge pipeline. Monitor it to catch long-running merges, track TTL processing, and diagnose part accumulation issues before they impact insert performance.
