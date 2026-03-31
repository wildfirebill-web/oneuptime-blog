# How to Move Data Between Tables Using TTL in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, TTL, Data Lifecycle, Storage Tier, MergeTree

Description: Learn how to use ClickHouse TTL rules to automatically move data between storage volumes and tables as it ages, implementing tiered data management.

---

## TTL Volume Moves vs Table Moves

ClickHouse TTL supports moving data to different storage volumes within the same table (transparent to queries) or physically detaching and attaching partitions to different tables. Volume moves are automatic; partition moves require orchestration.

## Setting Up Storage Volumes

Define hot and cold volumes in storage configuration:

```xml
<storage_configuration>
  <disks>
    <hot_disk>
      <type>local</type>
      <path>/var/lib/clickhouse/hot/</path>
    </hot_disk>
    <cold_disk>
      <type>s3</type>
      <endpoint>https://s3.amazonaws.com/my-bucket/clickhouse/</endpoint>
    </cold_disk>
  </disks>
  <policies>
    <tiered>
      <volumes>
        <hot><disk>hot_disk</disk></hot>
        <cold><disk>cold_disk</disk></cold>
      </volumes>
    </tiered>
  </policies>
</storage_configuration>
```

## Moving Data Between Volumes via TTL

```sql
CREATE TABLE events (
    timestamp   DateTime,
    user_id     UInt64,
    event_type  LowCardinality(String),
    payload     String
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(timestamp)
ORDER BY (timestamp, user_id)
TTL
    timestamp + INTERVAL 30 DAY TO VOLUME 'cold'
SETTINGS storage_policy = 'tiered';
```

After 30 days, ClickHouse automatically moves parts to the cold S3 volume during background merges.

## Verifying Data Location

```sql
SELECT
    name,
    disk_name,
    formatReadableSize(bytes_on_disk) AS size,
    min_time,
    max_time
FROM system.parts
WHERE table = 'events' AND active = 1
ORDER BY min_time;
```

Parts on hot disk will show `hot_disk`; moved parts will show `cold_disk`.

## Moving Data Between Physical Tables

For moving data to a completely different table (e.g., an archive table with different schema):

```sql
-- 1. Export old data to archive table
INSERT INTO events_archive
SELECT * FROM events WHERE toDate(timestamp) < '2024-01-01';

-- 2. Delete from source after confirming
ALTER TABLE events DELETE WHERE toDate(timestamp) < '2024-01-01';
```

Automate this with a scheduled job:

```sql
CREATE SCHEDULED JOB move_old_events
    SCHEDULE '0 3 * * *'
AS
    INSERT INTO events_archive
    SELECT * FROM events
    WHERE timestamp < now() - INTERVAL 365 DAY;
```

## Partition-Level Move

The most efficient bulk move for large datasets is partition attachment:

```sql
-- Detach old partition from source
ALTER TABLE events DETACH PARTITION '202301';

-- Attach to archive table (must have compatible schema)
ALTER TABLE events_archive ATTACH PARTITION '202301';
```

## Monitoring TTL Move Progress

```sql
SELECT
    table,
    reason,
    count() AS operations,
    sum(bytes_on_disk) AS bytes_moved
FROM system.part_log
WHERE reason IN ('TTLDeleteMerge', 'TTLMoveMerge')
  AND event_date = today()
GROUP BY table, reason;
```

## Summary

ClickHouse TTL volume moves are the recommended approach for transparent, automatic data tiering within a single table. Use partition-level DETACH/ATTACH for bulk moves between tables at minimal I/O cost. Combine both strategies with storage policies to implement a full hot-warm-cold data lifecycle without external orchestration.
