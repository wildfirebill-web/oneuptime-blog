# How to Configure ClickHouse Part Check Intervals

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Part Check, Data Integrity, Replication, Configuration, MergeTree

Description: Configure ClickHouse part check intervals to balance data integrity verification with background resource usage on replicated MergeTree tables.

---

## What Are Part Checks?

ClickHouse periodically checks the integrity of data parts on disk. Each part is a directory containing column data files and checksums. Part checks verify that stored data matches the recorded checksums, catching silent data corruption or disk errors early.

## Relevant Settings

```xml
<!-- config.xml -->
<check_delay_period>5</check_delay_period>
<cleanup_delay_period>30</cleanup_delay_period>
<merge_selecting_sleep_ms>5000</merge_selecting_sleep_ms>
```

In newer versions, you can configure via `MergeTree` settings:

```sql
ALTER TABLE my_table MODIFY SETTING
    check_sample_column_probability = 0.1;
```

## Part Check Queue

For replicated tables, parts are checked and reconciled via the replication log:

```sql
SELECT
    table,
    type,
    create_time,
    required_quorum,
    source_replica
FROM system.replication_queue
WHERE type = 'CHECK_PART'
ORDER BY create_time DESC
LIMIT 20;
```

## Triggering Manual Part Checks

Force a check on a specific table:

```sql
CHECK TABLE my_table;
```

Check a specific partition:

```sql
CHECK TABLE my_table PARTITION '2026-03';
```

The output shows each part and whether it passed or failed:

```text
part_name            | is_ok | message
20260101_1_1000_5    | 1     |
20260101_1001_2000_5 | 0     | Checksum mismatch
```

## Handling Failed Parts

If a part fails the check:

1. On a non-replicated table, detach the corrupt part and restore from backup:

```sql
ALTER TABLE my_table DETACH PART '20260101_1001_2000_5';
```

2. On a replicated table, ClickHouse automatically fetches the correct part from another replica:

```sql
SYSTEM SYNC REPLICA my_table;
```

## Configuring Check Frequency

The check delay controls how often ClickHouse scans for parts needing verification:

```xml
<check_delay_period>30</check_delay_period>
```

On high-write clusters with many parts, increase this to reduce background IO:

```xml
<check_delay_period>120</check_delay_period>
```

On clusters with a history of disk issues, keep it low for faster detection:

```xml
<check_delay_period>5</check_delay_period>
```

## Monitoring Part Health

Track recently checked parts and errors:

```sql
SELECT
    table,
    count() AS parts,
    countIf(is_currently_merging) AS merging,
    countIf(bytes_on_disk > 0) AS healthy
FROM system.parts
WHERE active AND database = 'default'
GROUP BY table;
```

## Summary

Configure ClickHouse part check intervals by balancing check frequency against background IO load. On replicated clusters, rely on automatic part re-fetching to recover from corruption. Use `CHECK TABLE` to manually verify specific tables or partitions, and monitor `system.replication_queue` for pending check operations.
