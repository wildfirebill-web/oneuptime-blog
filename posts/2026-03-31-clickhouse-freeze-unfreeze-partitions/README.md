# How to Freeze and Unfreeze Partitions in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, DDL, ALTER TABLE, Partition, Backup, Freeze

Description: Learn how to use ALTER TABLE FREEZE PARTITION and UNFREEZE in ClickHouse to create consistent partition snapshots for backups using the shadow directory.

---

`ALTER TABLE FREEZE PARTITION` creates a consistent, point-in-time snapshot of a partition's data files using hard links. The hard-linked files are placed in ClickHouse's `shadow/` directory and do not consume additional disk space (only metadata) as long as the original files remain. This is one of the most reliable ways to create a low-impact, crash-consistent backup of specific partitions without pausing ingestion.

## How FREEZE Works

When you freeze a partition, ClickHouse:

1. Acquires a brief lock on the partition.
2. Creates hard links to all active part files in a numbered subdirectory under `shadow/`.
3. Releases the lock.

The `shadow/` directory path is typically:

```text
/var/lib/clickhouse/shadow/<backup_number>/data/<database>/<table>/
```

Because hard links share the same inode, the backup files are immediately consistent and take negligible extra space. Files are only deleted from disk when both the original and all hard links are removed.

## FREEZE PARTITION Syntax

Freeze a specific partition:

```sql
ALTER TABLE events
    FREEZE PARTITION '2024-01';
```

Freeze all partitions (entire table):

```sql
ALTER TABLE events
    FREEZE;
```

### WITH NAME Clause

Assign a named backup instead of using an auto-incremented number. This makes it easier to manage multiple backups:

```sql
ALTER TABLE events
    FREEZE PARTITION '2024-01'
    WITH NAME 'backup_jan_2024';
```

The snapshot is placed in:

```text
/var/lib/clickhouse/shadow/backup_jan_2024/data/default/events/
```

## UNFREEZE Syntax

Remove the hard-linked snapshot files from the `shadow/` directory:

```sql
ALTER TABLE events
    UNFREEZE WITH NAME 'backup_jan_2024';
```

This frees the shadow directory entries. The original active parts are unaffected.

## Listing Frozen Snapshots

ClickHouse does not currently expose frozen snapshots through a system table. List them at the OS level:

```bash
ls /var/lib/clickhouse/shadow/
```

To see which partitions are inside a named snapshot:

```bash
ls /var/lib/clickhouse/shadow/backup_jan_2024/data/default/events/
```

## Complete Backup Workflow

The freeze-copy-unfreeze pattern is a standard approach for partition-level backups:

```sql
-- Step 1: Freeze the target partition
ALTER TABLE events
    FREEZE PARTITION 202401
    WITH NAME 'backup_events_202401';
```

```bash
# Step 2: Copy the shadow directory to backup storage (rsync, s3cmd, rclone, etc.)
rsync -av \
  /var/lib/clickhouse/shadow/backup_events_202401/ \
  backup-server:/backups/clickhouse/events/202401/
```

```sql
-- Step 3: Remove the shadow snapshot after successful transfer
ALTER TABLE events
    UNFREEZE WITH NAME 'backup_events_202401';
```

## Restoring from a Freeze Backup

To restore a frozen partition, copy the part directories back into the table's `detached/` directory and then attach them:

```bash
# Copy parts from backup storage back to detached/
cp -r /backups/clickhouse/events/202401/* \
      /var/lib/clickhouse/data/default/events/detached/
```

```sql
-- Attach the restored parts
ALTER TABLE events
    ATTACH PARTITION 202401;
```

ClickHouse verifies checksums on attach. If verification succeeds, the data is immediately available.

## Full Table Freeze Example

```sql
CREATE TABLE sales
(
    order_id   UInt64,
    product_id UInt32,
    amount     Float64,
    sold_at    DateTime
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(sold_at)
ORDER BY (product_id, sold_at);

-- Insert sample data across two months
INSERT INTO sales VALUES
    (1, 101, 29.99, '2024-01-15 10:00:00'),
    (2, 202, 49.99, '2024-02-20 14:30:00');

-- Freeze all partitions with a timestamped backup name
ALTER TABLE sales
    FREEZE
    WITH NAME 'sales_full_20240301';

-- Confirm shadow directory was created
-- (run in shell)
-- ls /var/lib/clickhouse/shadow/sales_full_20240301/data/default/sales/

-- After backup is confirmed, unfreeze
ALTER TABLE sales
    UNFREEZE WITH NAME 'sales_full_20240301';
```

## ON CLUSTER for Distributed Setups

In a distributed cluster, freeze must be run on each replica independently because each node holds its own data files:

```sql
ALTER TABLE events ON CLUSTER '{cluster}'
    FREEZE PARTITION 202401
    WITH NAME 'backup_jan_2024';
```

Each node creates its own shadow directory. Copy each node's snapshot to backup storage separately, maintaining the replica-to-backup mapping for accurate restoration.

## Key Differences from clickhouse-backup Tools

`FREEZE PARTITION` is a low-level primitive. Tools like `clickhouse-backup` use freeze under the hood but add:
- Metadata backup (schema DDL).
- Automated upload to object storage (S3, GCS, Azure Blob).
- Centralized restore workflows.

For production environments, using a dedicated backup tool built on top of freeze is recommended.

## Summary

`ALTER TABLE FREEZE PARTITION` creates instant, space-efficient hard-link snapshots in the `shadow/` directory, making it ideal for partition-level backups with minimal performance impact. Use `WITH NAME` to assign meaningful backup identifiers, copy the shadow directory to durable storage, and then `UNFREEZE` to clean up. Restoration involves copying parts back into the `detached/` directory and issuing `ATTACH PARTITION`.
