# How to Use TTL Policies to Manage ClickHouse Storage Costs

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, TTL, Storage, Cost Optimization, Data Lifecycle

Description: Learn how to use ClickHouse TTL policies to automatically delete, move, or aggregate old data to control storage growth and reduce costs.

---

## What Is TTL in ClickHouse?

TTL (time-to-live) in ClickHouse defines rules for what happens to data after a certain age. You can delete rows, move them to cheaper storage tiers, or aggregate them into summaries. TTL rules are applied automatically during background merges.

## Basic Row Deletion TTL

Delete rows older than 90 days:

```sql
ALTER TABLE events
    MODIFY TTL created_at + INTERVAL 90 DAY;
```

ClickHouse applies this during background merges. To force immediate execution:

```sql
OPTIMIZE TABLE events FINAL;
```

## Column-Level TTL

Instead of deleting entire rows, set a specific column to its default value after a time period:

```sql
ALTER TABLE user_sessions
    MODIFY COLUMN session_data String TTL created_at + INTERVAL 30 DAY;
```

After 30 days, `session_data` is reset to an empty string but the row remains.

## Tiered Storage: Move to Cold Disk

Define a storage policy in `storage_policy.xml`:

```xml
<storage_configuration>
  <disks>
    <hot_disk>
      <path>/var/lib/clickhouse/data/hot/</path>
    </hot_disk>
    <cold_disk>
      <path>/mnt/cold-storage/clickhouse/</path>
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

Apply the policy and TTL:

```sql
ALTER TABLE events
    MODIFY TTL
        created_at + INTERVAL 7 DAY TO DISK 'cold_disk',
        created_at + INTERVAL 365 DAY DELETE;
```

Data moves to cold disk after 7 days and is deleted after 1 year.

## Tiered Storage to S3

```xml
<s3_cold>
  <type>s3</type>
  <endpoint>https://s3.amazonaws.com/your-bucket/clickhouse/</endpoint>
  <access_key_id>AK...</access_key_id>
  <secret_access_key>...</secret_access_key>
</s3_cold>
```

```sql
ALTER TABLE events
    MODIFY TTL
        created_at + INTERVAL 30 DAY TO DISK 's3_cold',
        created_at + INTERVAL 2 YEAR DELETE;
```

## Aggregating Old Data Instead of Deleting

Replace high-resolution old data with summaries:

```sql
-- Keep 1-minute granularity for 7 days, then aggregate to hourly
CREATE TABLE events_hourly AS events
ENGINE = AggregatingMergeTree()
ORDER BY (event_type, toStartOfHour(created_at));

-- TTL on source table moves to aggregated form
ALTER TABLE events
    MODIFY TTL created_at + INTERVAL 7 DAY TO TABLE events_hourly;
```

## Viewing Current TTL Settings

```sql
SELECT
    table,
    ttl_info.columns AS column_ttls,
    ttl_info.table AS table_ttl
FROM system.tables
WHERE database = 'default'
  AND table = 'events';
```

## Checking TTL Execution

```sql
SELECT
    database,
    table,
    partition,
    min_ttl_delete,
    max_ttl_delete
FROM system.parts
WHERE active = 1
  AND table = 'events'
ORDER BY min_ttl_delete
LIMIT 10;
```

## Summary

ClickHouse TTL policies automate data lifecycle management: delete rows on a schedule, reset column values, move data to cheaper storage tiers, or aggregate old rows into summary tables. Combine row deletion TTL with S3-backed cold storage to achieve an efficient hot-warm-cold architecture that controls costs without losing access to historical data.
