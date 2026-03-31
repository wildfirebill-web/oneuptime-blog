# How to Configure ClickHouse Multi-Disk Storage

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Storage, Database, Configuration, Performance

Description: Learn how to configure multiple storage disks in ClickHouse to spread data across different volumes and devices for better performance and capacity.

---

ClickHouse can be configured to store data across multiple physical or logical disks. This lets you combine fast NVMe drives with large capacity HDDs, separate hot data from cold data, or simply span a table across several volumes. The configuration lives in `config.xml` (or a drop-in file under `config.d/`) and is applied through storage policies.

## How ClickHouse Storage Is Structured

The storage hierarchy in ClickHouse works from bottom to top:

- **Disk** - a named mount point or object storage endpoint
- **Volume** - an ordered group of disks
- **Storage policy** - an ordered list of volumes with optional move conditions

When ClickHouse writes a new data part it places it on the first volume in the active storage policy. Parts can be moved to subsequent volumes based on age, size, or TTL rules.

## Declaring Disks in Configuration

Add a `storage_configuration` section to `config.xml` or create `/etc/clickhouse-server/config.d/storage.xml`:

```xml
<clickhouse>
  <storage_configuration>
    <disks>
      <!-- default disk already exists, but you can override it -->
      <default>
        <path>/var/lib/clickhouse/</path>
      </default>

      <!-- fast NVMe drive -->
      <nvme>
        <path>/mnt/nvme/clickhouse/</path>
      </nvme>

      <!-- large capacity HDD -->
      <hdd>
        <path>/mnt/hdd/clickhouse/</path>
      </hdd>
    </disks>
  </storage_configuration>
</clickhouse>
```

Each `<path>` must exist on the filesystem and be writable by the `clickhouse` user before the server starts.

## Grouping Disks into Volumes and Policies

Extend the same configuration block with `<policies>`:

```xml
<clickhouse>
  <storage_configuration>
    <disks>
      <default>
        <path>/var/lib/clickhouse/</path>
      </default>
      <nvme>
        <path>/mnt/nvme/clickhouse/</path>
      </nvme>
      <hdd>
        <path>/mnt/hdd/clickhouse/</path>
      </hdd>
    </disks>

    <policies>
      <tiered_policy>
        <volumes>
          <hot>
            <disk>nvme</disk>
          </hot>
          <warm>
            <disk>default</disk>
          </warm>
          <cold>
            <disk>hdd</disk>
          </cold>
        </volumes>
      </tiered_policy>
    </policies>
  </storage_configuration>
</clickhouse>
```

## Applying a Storage Policy to a Table

Reference the policy by name in the `SETTINGS` clause when creating a table:

```sql
CREATE TABLE events
(
    event_id   UInt64,
    user_id    UInt32,
    event_type LowCardinality(String),
    payload    String,
    created_at DateTime
)
ENGINE = MergeTree()
ORDER BY (created_at, event_id)
SETTINGS storage_policy = 'tiered_policy';
```

To change the policy on an existing table:

```sql
ALTER TABLE events
    MODIFY SETTING storage_policy = 'tiered_policy';
```

You cannot switch to a policy that excludes disks already used by the table.

## Verifying Disk and Volume Assignments

Query the system tables to inspect what is configured:

```sql
-- List all configured disks
SELECT name, path, free_space, total_space, type
FROM system.disks;

-- List all policies and their volumes
SELECT policy_name, volume_name, disks, volume_priority, max_data_part_size
FROM system.storage_policies;

-- See which disk each active part lives on
SELECT
    table,
    name        AS part_name,
    disk_name,
    formatReadableSize(bytes_on_disk) AS size_on_disk
FROM system.parts
WHERE active = 1
  AND database = currentDatabase()
ORDER BY table, disk_name;
```

## Controlling New Part Placement

By default ClickHouse places all new parts on the first volume. To distribute writes round-robin across disks within a single volume, set `load_balancing`:

```xml
<tiered_policy>
  <volumes>
    <hot>
      <disk>nvme1</disk>
      <disk>nvme2</disk>
      <load_balancing>round_robin</load_balancing>
    </hot>
  </volumes>
</tiered_policy>
```

You can also reserve a volume for parts below a certain size:

```xml
<hot>
  <disk>nvme</disk>
  <max_data_part_size_bytes>1073741824</max_data_part_size_bytes>
</hot>
```

Parts larger than 1 GiB will skip the hot volume and land on the next one.

## Moving Parts Manually

Force a specific part to a different disk immediately:

```sql
ALTER TABLE events
    MOVE PART '20240101_1_1_0' TO DISK 'hdd';
```

Move all parts of a partition:

```sql
ALTER TABLE events
    MOVE PARTITION '2024-01-01' TO VOLUME 'cold';
```

## Preparing Disk Paths

Run the following on the server before starting ClickHouse:

```bash
# Create mount point directories
mkdir -p /mnt/nvme/clickhouse
mkdir -p /mnt/hdd/clickhouse

# Set ownership
chown -R clickhouse:clickhouse /mnt/nvme/clickhouse
chown -R clickhouse:clickhouse /mnt/hdd/clickhouse

# Verify
ls -la /mnt/nvme/
ls -la /mnt/hdd/
```

## Common Mistakes

- Forgetting to create the directory before starting ClickHouse causes startup failure.
- Assigning the same path to two different disk names produces unpredictable behavior.
- Switching a table to a policy that lacks one of its current disks is rejected by ClickHouse.

## Summary

Multi-disk storage in ClickHouse is configured through a three-layer model: disks, volumes, and policies. Declare each disk path in `storage_configuration`, group them into volumes inside a policy, and assign the policy to a table with `SETTINGS storage_policy`. Use `system.disks`, `system.storage_policies`, and `system.parts` to verify placement and troubleshoot issues.
