# How to Use storage_configuration in ClickHouse Config

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Storage, Configuration, Disk, Volume, Policy

Description: Master the storage_configuration section in ClickHouse config to define disks, volumes, and storage policies for automatic data placement and tiering across storage backends.

---

## Introduction

The `storage_configuration` section in `config.xml` is the central place to define all storage resources in ClickHouse. It has three nested sections: `disks`, `policies`, and their `volumes`. Together they control where data parts land when written and how parts migrate between storage tiers automatically.

## Configuration Structure

```mermaid
graph TD
    A[storage_configuration] --> B[disks]
    A --> C[policies]
    B --> D[disk: default]
    B --> E[disk: ssd]
    B --> F[disk: s3_cold]
    C --> G[policy: hot_to_cold]
    G --> H[volume: hot]
    G --> I[volume: cold]
    H --> D
    H --> E
    I --> F
```

## Minimal Example

Add a file at `/etc/clickhouse-server/config.d/storage.xml`:

```xml
<clickhouse>
  <storage_configuration>

    <!-- 1. Define disks -->
    <disks>
      <ssd>
        <type>local</type>
        <path>/mnt/ssd/clickhouse/</path>
      </ssd>
      <s3_cold>
        <type>s3</type>
        <endpoint>https://s3.amazonaws.com/my-bucket/clickhouse/</endpoint>
        <access_key_id>AKIAIOSFODNN7EXAMPLE</access_key_id>
        <secret_access_key>wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY</secret_access_key>
      </s3_cold>
    </disks>

    <!-- 2. Define storage policies -->
    <policies>
      <hot_to_cold>
        <volumes>
          <hot>
            <disk>ssd</disk>
            <max_data_part_size_bytes>10737418240</max_data_part_size_bytes>
          </hot>
          <cold>
            <disk>s3_cold</disk>
            <perform_ttl_move_on_insert>true</perform_ttl_move_on_insert>
          </cold>
        </volumes>
        <move_factor>0.2</move_factor>
      </hot_to_cold>
    </policies>

  </storage_configuration>
</clickhouse>
```

## Key Fields Explained

### Disk Fields

| Field | Description |
|---|---|
| `type` | Disk backend: `local`, `s3`, `azure_blob_storage`, `cache`, `encrypted` |
| `path` | Local filesystem path (for local disks) |
| `endpoint` | S3/GCS endpoint URL |
| `access_key_id` | S3 access key |
| `secret_access_key` | S3 secret key |
| `send_metadata` | Attach ClickHouse metadata to S3 objects |

### Volume Fields

| Field | Description |
|---|---|
| `disk` | One or more disk references |
| `max_data_part_size_bytes` | Parts larger than this go to the next volume |
| `perform_ttl_move_on_insert` | Move parts immediately on TTL match at insert time |

### Policy Fields

| Field | Description |
|---|---|
| `move_factor` | Free-space threshold (0.0-1.0) that triggers part moves to next volume |

## JBOD Volume (Multiple Disks Round-Robin)

Place multiple local disks in one volume for parallel writes:

```xml
<volumes>
  <jbod_hot>
    <disk>ssd1</disk>
    <disk>ssd2</disk>
    <disk>ssd3</disk>
  </jbod_hot>
</volumes>
```

## Applying a Policy to a Table

```sql
CREATE TABLE events
(
    event_id   UInt64,
    event_time DateTime,
    payload    String
)
ENGINE = MergeTree
ORDER BY event_time
SETTINGS storage_policy = 'hot_to_cold';
```

## Changing a Table's Policy

```sql
ALTER TABLE events MODIFY SETTING storage_policy = 'hot_to_cold';
```

## Verifying Policies and Volumes

```sql
SELECT policy_name, volume_name, disks, volume_priority
FROM system.storage_policies;
```

```sql
SELECT name, type, path, free_space
FROM system.disks;
```

## TTL-Based Moves

Combine storage policies with TTL rules to automatically move old data to cold storage:

```sql
ALTER TABLE events
    MODIFY TTL event_time + INTERVAL 30 DAY
    TO VOLUME 'cold';
```

## Reloading Config

After editing `storage.xml`, reload without restart:

```sql
SYSTEM RELOAD CONFIG;
```

## Summary

The `storage_configuration` section lets you define named disks (local, S3, GCS, Azure, cache, encrypted), group them into volumes, and compose volumes into storage policies. Tables reference policies by name. ClickHouse then automatically routes new data parts to the correct disk and migrates parts between volumes based on `max_data_part_size_bytes`, `move_factor`, and TTL rules.
