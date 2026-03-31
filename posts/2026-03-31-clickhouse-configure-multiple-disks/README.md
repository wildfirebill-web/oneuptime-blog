# How to Configure Multiple Disks in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Storage, Disk, Configuration, MergeTree

Description: Learn how to configure multiple disks in ClickHouse using storage policies to distribute data across different storage volumes and tiers.

---

ClickHouse's multi-disk storage configuration lets you use multiple physical disks or storage backends - SSD, HDD, NVMe, and object storage - within a single cluster. Data is distributed and tiered according to policies you define.

## Disk Configuration Basics

Configure disks in `config.d/storage.xml`:

```xml
<clickhouse>
    <storage_configuration>
        <disks>
            <!-- Fast NVMe SSD -->
            <nvme_ssd>
                <type>local</type>
                <path>/data/nvme/clickhouse/</path>
            </nvme_ssd>

            <!-- Standard SSD -->
            <sata_ssd>
                <type>local</type>
                <path>/data/ssd/clickhouse/</path>
            </sata_ssd>

            <!-- HDD for cold data -->
            <hdd>
                <type>local</type>
                <path>/data/hdd/clickhouse/</path>
            </hdd>
        </disks>
    </storage_configuration>
</clickhouse>
```

Ensure the directories exist and are owned by the ClickHouse user:

```bash
mkdir -p /data/nvme/clickhouse /data/ssd/clickhouse /data/hdd/clickhouse
chown -R clickhouse:clickhouse /data/
```

## Defining Volumes and Storage Policies

Group disks into volumes, then combine volumes into policies:

```xml
<policies>
    <tiered_storage>
        <volumes>
            <hot>
                <disk>nvme_ssd</disk>
                <max_data_part_size_bytes>10737418240</max_data_part_size_bytes>
            </hot>
            <warm>
                <disk>sata_ssd</disk>
                <max_data_part_size_bytes>107374182400</max_data_part_size_bytes>
            </warm>
            <cold>
                <disk>hdd</disk>
            </cold>
        </volumes>
        <move_factor>0.2</move_factor>
    </tiered_storage>
</policies>
```

## Applying a Storage Policy to a Table

```sql
CREATE TABLE events
(
    event_time DateTime,
    user_id UInt64,
    event_type String
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_time)
ORDER BY (event_time, user_id)
SETTINGS storage_policy = 'tiered_storage';
```

Or add a policy to an existing table:

```sql
ALTER TABLE events MODIFY SETTING storage_policy = 'tiered_storage';
```

## Checking Current Disk Assignment

Verify which disk holds each part:

```sql
SELECT
    table,
    partition,
    disk_name,
    count() AS part_count,
    formatReadableSize(sum(bytes_on_disk)) AS size
FROM system.parts
WHERE active = 1 AND database = 'production'
GROUP BY table, partition, disk_name
ORDER BY table, partition, disk_name;
```

## Checking Disk Space

Monitor free space across all configured disks:

```sql
SELECT
    name AS disk_name,
    path,
    formatReadableSize(free_space) AS free_space,
    formatReadableSize(total_space) AS total_space,
    round((total_space - free_space) / total_space * 100, 1) AS used_pct
FROM system.disks
ORDER BY name;
```

## Moving Data Between Disks

Manually move parts to a specific disk:

```sql
-- Move all parts of a partition to a specific volume
ALTER TABLE events MOVE PARTITION '202601' TO VOLUME 'cold';

-- Move specific part
ALTER TABLE events MOVE PART 'all_0_1_1' TO DISK 'hdd';
```

## Summary

ClickHouse multi-disk configuration involves defining named disks in `storage_configuration`, grouping them into volumes with optional size limits, combining volumes into storage policies, and applying policies to tables. Use `system.disks` and `system.parts` to monitor disk usage and part placement. Storage policies enable automatic data tiering from fast to slow storage as parts age.
