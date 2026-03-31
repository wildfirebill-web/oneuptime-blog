# How to Configure Hot/Warm/Cold Storage Policies in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Storage, Tiering, Hot Warm Cold, Performance

Description: Learn how to configure hot, warm, and cold storage tiers in ClickHouse to automatically move data from fast to slow storage as it ages.

---

Storage tiering in ClickHouse automatically moves data from expensive fast storage to cheaper slower storage as it ages. New writes go to SSD (hot), recent data stays on standard SSD (warm), and old data migrates to HDD or S3 (cold).

## Architecture Overview

A typical tiered storage setup:

- **Hot tier** - NVMe SSD, high IOPS, most recent data (last 7 days)
- **Warm tier** - Standard SATA SSD, recent data (last 30-90 days)
- **Cold tier** - HDD or S3, historical data (older than 90 days)

## Full Tiered Storage Configuration

```xml
<clickhouse>
    <storage_configuration>
        <disks>
            <hot_nvme>
                <type>local</type>
                <path>/data/nvme/</path>
            </hot_nvme>
            <warm_ssd>
                <type>local</type>
                <path>/data/ssd/</path>
            </warm_ssd>
            <cold_s3>
                <type>s3</type>
                <endpoint>https://s3.us-east-1.amazonaws.com/my-bucket/cold/</endpoint>
                <access_key_id>ACCESS_KEY</access_key_id>
                <secret_access_key>SECRET_KEY</secret_access_key>
            </cold_s3>
        </disks>
        <policies>
            <hot_warm_cold>
                <volumes>
                    <hot>
                        <disk>hot_nvme</disk>
                        <max_data_part_size_bytes>5368709120</max_data_part_size_bytes>
                    </hot>
                    <warm>
                        <disk>warm_ssd</disk>
                        <max_data_part_size_bytes>53687091200</max_data_part_size_bytes>
                    </warm>
                    <cold>
                        <disk>cold_s3</disk>
                    </cold>
                </volumes>
                <move_factor>0.1</move_factor>
            </hot_warm_cold>
        </policies>
    </storage_configuration>
</clickhouse>
```

## Using TTL for Automatic Data Movement

TTL rules trigger data movement between volumes based on age:

```sql
CREATE TABLE events
(
    event_time DateTime,
    user_id UInt64,
    data String
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_time)
ORDER BY (event_time, user_id)
TTL
    event_time + INTERVAL 7 DAY TO VOLUME 'warm',
    event_time + INTERVAL 90 DAY TO VOLUME 'cold'
SETTINGS storage_policy = 'hot_warm_cold';
```

Data written to the `hot` volume automatically migrates to `warm` after 7 days and to `cold` after 90 days.

## Adding TTL Moves to Existing Tables

```sql
ALTER TABLE events
MODIFY TTL
    event_time + INTERVAL 7 DAY TO VOLUME 'warm',
    event_time + INTERVAL 90 DAY TO VOLUME 'cold';
```

## Monitoring Data Tier Distribution

See how data is distributed across tiers:

```sql
SELECT
    disk_name,
    count() AS parts,
    formatReadableSize(sum(bytes_on_disk)) AS size,
    min(min_time) AS oldest_data,
    max(max_time) AS newest_data
FROM system.parts
WHERE active = 1 AND table = 'events'
GROUP BY disk_name
ORDER BY disk_name;
```

## Configuring move_factor

The `move_factor` controls when ClickHouse starts moving data to the next tier. With `move_factor = 0.1`, ClickHouse moves data when a volume reaches 90% full:

```xml
<move_factor>0.1</move_factor>
```

Set lower values (0.05) for more aggressive proactive moves before disks fill up.

## Checking TTL Move Status

```sql
-- Check pending TTL moves
SELECT
    database,
    table,
    partition,
    disk_name
FROM system.parts
WHERE active = 1
  AND modification_time < now() - INTERVAL 7 DAY
  AND disk_name = 'hot_nvme'
ORDER BY modification_time;
```

Trigger TTL processing if moves are delayed:

```sql
OPTIMIZE TABLE events FINAL;
SYSTEM START TTL MERGES;
```

## Summary

Hot/warm/cold storage policies in ClickHouse use volume-based storage policies combined with TTL rules to automatically move data between tiers as it ages. Configure disks for each tier, define volumes with size limits, and use TTL TO VOLUME rules to control when data migrates. Monitor tier distribution via `system.parts` to verify data is moving as expected.
