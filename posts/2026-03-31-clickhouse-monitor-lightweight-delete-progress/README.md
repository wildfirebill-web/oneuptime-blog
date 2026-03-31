# How to Monitor Lightweight Delete Progress in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Lightweight Delete, Monitoring, system.parts, Merge, Performance

Description: Learn how to monitor the progress of lightweight deletes in ClickHouse by tracking delete masks and merge activity using system tables.

---

Lightweight DELETE in ClickHouse creates delete masks immediately but relies on background merges to physically remove the data. Monitoring this process helps you understand disk usage, read performance, and when cleanup has completed.

## Step 1 - Confirm the Delete Was Applied

After running a lightweight DELETE, verify the rows are logically gone:

```sql
DELETE FROM events WHERE event_date < '2024-01-01';

SELECT count() FROM events WHERE event_date < '2024-01-01';
-- Should return 0
```

## Step 2 - Count Parts with Pending Delete Masks

```sql
SELECT
    database,
    table,
    countIf(has_lightweight_delete) AS parts_with_masks,
    count() AS total_active_parts
FROM system.parts
WHERE active = 1 AND table = 'events'
GROUP BY database, table;
```

When `parts_with_masks` drops to 0, all physical cleanup is done.

## Step 3 - Track Merge Activity

ClickHouse merges parts in the background, which is what removes the masked rows. Watch merge progress:

```sql
SELECT
    database,
    table,
    elapsed,
    progress,
    num_parts,
    result_part_name
FROM system.merges
WHERE table = 'events'
ORDER BY elapsed DESC;
```

## Step 4 - Estimate Disk Still Used by Deleted Rows

Unfortunately ClickHouse does not expose the exact bytes behind delete masks, but you can compare total rows to logical rows:

```sql
SELECT
    sum(rows) AS physical_rows,
    formatReadableSize(sum(bytes_on_disk)) AS disk_used
FROM system.parts
WHERE table = 'events' AND active = 1 AND has_lightweight_delete = 1;
```

## Step 5 - Force Cleanup with OPTIMIZE

If you need to reclaim disk space sooner, trigger a merge manually:

```sql
OPTIMIZE TABLE events FINAL;
```

Then re-check:

```sql
SELECT count()
FROM system.parts
WHERE table = 'events' AND active = 1 AND has_lightweight_delete = 1;
-- Should be 0
```

## Automating Cleanup Monitoring

You can build a simple monitoring query to alert when masked parts exceed a threshold:

```sql
SELECT
    table,
    countIf(has_lightweight_delete) AS masked_parts
FROM system.parts
WHERE active = 1
GROUP BY table
HAVING masked_parts > 10;
```

Feed this into your alerting system to trigger an `OPTIMIZE` when needed.

## Summary

Monitoring lightweight delete progress means watching `system.parts` for `has_lightweight_delete = 1` and `system.merges` for background activity. Once all parts are merged and masks are gone, physical cleanup is complete. Use `OPTIMIZE TABLE FINAL` to accelerate the process when disk reclamation is urgent.
