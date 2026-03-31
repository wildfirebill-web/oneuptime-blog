# How to Configure ClickHouse Storage Policies

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Storage, Database, Configuration, Administration

Description: Learn how to define and apply ClickHouse storage policies to control where table data is written, how it moves between tiers, and how capacity limits are enforced.

---

A storage policy is the central abstraction in ClickHouse's storage system. It ties together named disks into ordered volumes and expresses rules about how data flows between them. Every MergeTree table uses exactly one storage policy. Understanding how to design and apply storage policies lets you manage data placement, capacity, and performance precisely.

## Storage Policy Concepts

- **Disk** - a named path on the local filesystem or a reference to object storage
- **Volume** - an ordered list of disks; new parts land here
- **Policy** - an ordered list of volumes; defines the full data lifecycle

ClickHouse writes new parts to the first volume in the policy. When a volume fills up (based on `move_factor`) or a TTL rule fires, parts move to the next volume in order.

## Basic Single-Volume Policy

The built-in `default` policy has one volume and one disk:

```xml
<clickhouse>
  <storage_configuration>
    <disks>
      <default>
        <path>/var/lib/clickhouse/</path>
      </default>
    </disks>
    <policies>
      <default>
        <volumes>
          <default>
            <disk>default</disk>
          </default>
        </volumes>
      </default>
    </policies>
  </storage_configuration>
</clickhouse>
```

## Two-Volume Tiered Policy

```xml
<clickhouse>
  <storage_configuration>
    <disks>
      <fast>
        <path>/mnt/nvme/clickhouse/</path>
      </fast>
      <slow>
        <path>/mnt/hdd/clickhouse/</path>
      </slow>
    </disks>

    <policies>
      <tiered>
        <volumes>
          <!-- New parts land here first -->
          <hot>
            <disk>fast</disk>
            <!-- Parts larger than 2 GiB skip the hot volume -->
            <max_data_part_size_bytes>2147483648</max_data_part_size_bytes>
          </hot>
          <!-- Parts move here when hot is 80% full or after TTL -->
          <cold>
            <disk>slow</disk>
          </cold>
        </volumes>
        <!-- Start moving when hot volume is 20% free -->
        <move_factor>0.2</move_factor>
      </tiered>
    </policies>
  </storage_configuration>
</clickhouse>
```

## Three-Volume Policy with S3 Archive

```xml
<clickhouse>
  <storage_configuration>
    <disks>
      <nvme>
        <path>/mnt/nvme/clickhouse/</path>
      </nvme>
      <hdd>
        <path>/mnt/hdd/clickhouse/</path>
      </hdd>
      <s3_archive>
        <type>s3</type>
        <endpoint>https://s3.us-east-1.amazonaws.com/my-bucket/archive/</endpoint>
        <access_key_id from_env="AWS_ACCESS_KEY_ID"/>
        <secret_access_key from_env="AWS_SECRET_ACCESS_KEY"/>
        <metadata_path>/var/lib/clickhouse/disks/s3_archive/</metadata_path>
      </s3_archive>
    </disks>

    <policies>
      <three_tier>
        <volumes>
          <hot>
            <disk>nvme</disk>
          </hot>
          <warm>
            <disk>hdd</disk>
          </warm>
          <archive>
            <disk>s3_archive</disk>
          </archive>
        </volumes>
        <move_factor>0.2</move_factor>
      </three_tier>
    </policies>
  </storage_configuration>
</clickhouse>
```

Apply with TTL:

```sql
CREATE TABLE audit_log
(
    event_id UInt64,
    actor    String,
    action   String,
    ts       DateTime
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(ts)
ORDER BY (ts, event_id)
TTL
    ts + INTERVAL 7 DAY TO VOLUME 'warm',
    ts + INTERVAL 30 DAY TO VOLUME 'archive',
    ts + INTERVAL 365 DAY DELETE
SETTINGS storage_policy = 'three_tier';
```

## JBOD Volume in a Policy

Multiple disks in a single volume for striping:

```xml
<policies>
  <jbod>
    <volumes>
      <main>
        <disk>disk1</disk>
        <disk>disk2</disk>
        <disk>disk3</disk>
        <load_balancing>least_used</load_balancing>
      </main>
    </volumes>
  </jbod>
</policies>
```

## Applying and Changing Policies

Assign at table creation:

```sql
CREATE TABLE logs
(
    id  UInt64,
    msg String,
    ts  DateTime
)
ENGINE = MergeTree()
ORDER BY ts
SETTINGS storage_policy = 'tiered';
```

Change the policy on an existing table:

```sql
ALTER TABLE logs
    MODIFY SETTING storage_policy = 'three_tier';
```

You can only switch to a policy that includes all disks currently used by the table.

## Inspecting Policies and Volumes

```sql
-- All configured policies
SELECT policy_name, volume_name, disks, volume_priority, max_data_part_size
FROM system.storage_policies;

-- All configured disks
SELECT name, type, path, free_space, total_space
FROM system.disks;

-- Per-table policy assignment
SELECT name, storage_policy
FROM system.tables
WHERE database = currentDatabase();
```

## Controlling Part Size Limits Per Volume

Use `max_data_part_size_bytes` to prevent large parts from landing on a volume. Parts exceeding the limit are written to the next volume:

```xml
<hot>
  <disk>nvme</disk>
  <!-- Skip hot volume for parts larger than 1 GiB -->
  <max_data_part_size_bytes>1073741824</max_data_part_size_bytes>
</hot>
```

This is useful when your hot disk is small and should only hold frequently merged (small) parts.

## Prefer Larger Parts on Slow Volumes

For the opposite effect - only move fully merged large parts to cold - configure `max_data_part_size_bytes` on the hot volume to be very large and rely on TTL for movement:

```sql
ALTER TABLE logs
    MODIFY TTL ts + INTERVAL 14 DAY TO VOLUME 'cold';
```

## Summary

Storage policies in ClickHouse bind named disks into ordered volumes and express the full lifecycle of data in a table. Design policies to match your hardware: use single volumes for simple setups, add a cold tier for archival, use JBOD volumes for capacity pooling, and use S3 volumes for unlimited low-cost archiving. Apply policies to tables with `SETTINGS storage_policy` and combine with TTL for automatic tier migration.
