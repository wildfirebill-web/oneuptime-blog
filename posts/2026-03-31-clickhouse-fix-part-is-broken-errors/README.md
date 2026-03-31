# How to Fix 'Part is broken' Errors in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Data Corruption, Error, MergeTree, Troubleshooting

Description: Fix 'Part is broken' errors in ClickHouse by identifying corrupted parts, detaching them, and restoring from replicas or backups.

---

"Part is broken" errors in ClickHouse indicate that one or more data parts in a MergeTree table have failed checksum verification. This typically results from disk hardware failure, file system corruption, or an interrupted write.

## Check Which Parts Are Broken

```sql
SELECT
    database,
    table,
    name AS part_name,
    disk_name,
    path,
    bytes_on_disk,
    data_compressed_bytes,
    modification_time
FROM system.parts
WHERE table = 'my_table' AND broken = 1;
```

Also check the error log:

```bash
sudo grep -i "broken\|checksum" /var/log/clickhouse-server/clickhouse-server.err.log | tail -30
```

## Verify Disk Health

Before fixing ClickHouse, check the underlying disk:

```bash
sudo dmesg | grep -i "i/o error\|sector\|disk"
sudo fsck /dev/sdb1  # Check filesystem
sudo smartctl -a /dev/sdb  # Check SMART data
```

## Detach the Broken Part

Move the broken part to the detached folder to stop ClickHouse from complaining:

```sql
ALTER TABLE my_database.my_table DETACH PART 'broken_part_name';
```

Or detach the entire partition containing the broken part:

```sql
ALTER TABLE my_database.my_table DETACH PARTITION 'partition_id';
```

## Recover from Replica (Replicated Tables)

If you have a replicated setup, ClickHouse can fetch the part from another replica:

```sql
-- Force the replica to re-fetch all parts from peers
SYSTEM RESTART REPLICA my_database.my_table;
```

Or fetch a specific partition:

```sql
ALTER TABLE my_database.my_table
FETCH PARTITION 'partition_id'
FROM '/clickhouse/tables/shard1/my_database/my_table';
```

## Remove Broken Parts

After confirming the data is recovered from a replica, remove the broken detached parts:

```bash
sudo rm -rf /var/lib/clickhouse/data/my_database/my_table/detached/broken_part_name
```

## Recover from Backup

For non-replicated tables, restore the affected partition from backup:

```bash
# Using clickhouse-backup tool
clickhouse-backup restore --table my_database.my_table --partitions 2024-01 my_backup
```

## Prevent Future Corruption

Enable checksums verification on startup:

```xml
<!-- config.xml -->
<check_delay_period>60</check_delay_period>
```

Use reliable storage (RAID, ECC memory) and set up replication as the primary protection against data loss.

## Summary

"Part is broken" errors signal data corruption at the storage level. Start by checking disk health with SMART tools. Detach the broken parts, then recover from a replica using `SYSTEM RESTART REPLICA` or `FETCH PARTITION`. For non-replicated tables, restore from backup. Always run ClickHouse on replicated storage or with replication enabled to protect against hardware failures.
