# How to Fix 'No free disk space' Errors in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Disk Space, Error, Storage, Troubleshooting

Description: Fix 'No free disk space' errors in ClickHouse by identifying space consumers, running OPTIMIZE, configuring TTL, and expanding storage.

---

ClickHouse throws "No free disk space" errors when the data directory runs out of space during a merge, INSERT, or part creation. The error prevents further writes and can also block background merges, leading to a growing number of small parts.

## Check Disk Usage

```bash
df -h /var/lib/clickhouse
du -sh /var/lib/clickhouse/data/*/* | sort -rh | head -20
```

From within ClickHouse:

```sql
SELECT
    database,
    table,
    formatReadableSize(sum(bytes_on_disk)) AS size_on_disk,
    count() AS parts
FROM system.parts
WHERE active
GROUP BY database, table
ORDER BY sum(bytes_on_disk) DESC
LIMIT 20;
```

## Free Space Immediately

Drop old partitions to reclaim space quickly:

```sql
-- See partition sizes
SELECT partition, formatReadableSize(sum(bytes_on_disk)) AS size
FROM system.parts
WHERE table = 'my_table' AND active
GROUP BY partition
ORDER BY partition;

-- Drop old partitions
ALTER TABLE my_table DROP PARTITION '2024-01';
```

## Remove Detached Parts

Detached parts are not served but consume disk space:

```bash
du -sh /var/lib/clickhouse/data/my_database/my_table/detached/
```

```sql
-- List detached parts
SELECT * FROM system.detached_parts WHERE table = 'my_table';
```

Remove old detached data:

```bash
sudo rm -rf /var/lib/clickhouse/data/my_database/my_table/detached/*
```

## Configure TTL to Expire Old Data

Add a TTL policy to automatically delete old data:

```sql
ALTER TABLE my_table
  MODIFY TTL event_date + INTERVAL 90 DAY;
```

Force TTL application:

```sql
OPTIMIZE TABLE my_table FINAL;
-- or
ALTER TABLE my_table MATERIALIZE TTL;
```

## Expand Storage

Add a new disk and configure ClickHouse storage policies in `config.xml`:

```xml
<storage_configuration>
  <disks>
    <disk2>
      <path>/mnt/disk2/clickhouse/</path>
    </disk2>
  </disks>
  <policies>
    <default>
      <volumes>
        <main>
          <disk>default</disk>
          <disk>disk2</disk>
        </main>
      </volumes>
    </default>
  </policies>
</storage_configuration>
```

## Set a Disk Space Reserve

Prevent ClickHouse from using 100% of disk:

```xml
<keep_free_space_bytes>10737418240</keep_free_space_bytes>
```

## Summary

"No free disk space" in ClickHouse requires immediate action to drop old partitions or detached parts. Long-term, configure TTL policies to automatically expire old data, monitor disk usage via `system.parts`, set `keep_free_space_bytes` as a safety buffer, and expand storage capacity using ClickHouse's multi-disk storage policies.
