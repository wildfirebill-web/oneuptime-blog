# How to Set Up Incremental Backups in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Backup, Incremental Backup, Data Protection, Storage

Description: Learn how to configure and run incremental backups in ClickHouse to reduce storage costs and backup time compared to full backups.

---

Full backups of large ClickHouse databases can take hours and consume significant storage. Incremental backups solve this by only storing data that changed since the last backup, dramatically reducing both time and storage costs.

## How ClickHouse Incremental Backups Work

ClickHouse compares data parts between the current state and the base backup. Parts that exist in the base backup are referenced rather than copied. Only new or changed parts are written to the incremental backup destination.

## Setting Up a Backup Disk

Configure a local or object storage disk for backups in `config.d/backup.xml`:

```xml
<clickhouse>
    <storage_configuration>
        <disks>
            <backups>
                <type>local</type>
                <path>/backup/clickhouse/</path>
            </backups>
        </disks>
    </storage_configuration>
    <backups>
        <allowed_disk>backups</allowed_disk>
        <allowed_path>/</allowed_path>
    </backups>
</clickhouse>
```

## Creating the Initial Full Backup

Always start with a full backup as the base:

```sql
BACKUP DATABASE production
TO Disk('backups', 'full_backup_2026-03-28/');
```

Verify it completed:

```sql
SELECT id, status, start_time, end_time, total_size
FROM system.backups
WHERE start_time >= '2026-03-28'
ORDER BY start_time DESC
LIMIT 5;
```

## Creating Incremental Backups

Reference the full backup as the base to create an incremental backup:

```sql
BACKUP DATABASE production
TO Disk('backups', 'incremental_2026-03-29/')
SETTINGS base_backup = Disk('backups', 'full_backup_2026-03-28/');
```

Chain incrementals on top of previous incrementals:

```sql
BACKUP DATABASE production
TO Disk('backups', 'incremental_2026-03-31/')
SETTINGS base_backup = Disk('backups', 'incremental_2026-03-30/');
```

## Backup Schedule Script

Automate a weekly full plus daily incremental schedule:

```bash
#!/bin/bash
TODAY=$(date +%Y-%m-%d)
DOW=$(date +%u)
BACKUP_BASE="/backup/clickhouse"

if [ "$DOW" -eq 1 ]; then
    clickhouse-client --query "BACKUP DATABASE production TO Disk('backups', 'full_${TODAY}/')"
    echo "$TODAY" > "$BACKUP_BASE/latest_full.txt"
else
    LATEST_FULL=$(cat "$BACKUP_BASE/latest_full.txt")
    YESTERDAY=$(date -d "yesterday" +%Y-%m-%d)
    PREV_BACKUP="incremental_${YESTERDAY}"
    if [ ! -d "$BACKUP_BASE/$PREV_BACKUP" ]; then
        PREV_BACKUP="full_${LATEST_FULL}"
    fi
    clickhouse-client --query "BACKUP DATABASE production TO Disk('backups', 'incremental_${TODAY}/') SETTINGS base_backup = Disk('backups', '${PREV_BACKUP}/')"
fi
```

## Restoring from an Incremental Backup

ClickHouse automatically resolves the backup chain during restore:

```sql
RESTORE DATABASE production AS production_restored
FROM Disk('backups', 'incremental_2026-03-31/');
```

ClickHouse reads the chain back to the full backup automatically.

## Summary

ClickHouse incremental backups use the `base_backup` setting to reference unchanged data parts rather than copying them. Start with a weekly full backup, take daily incrementals on top, and restore by pointing to any incremental - ClickHouse resolves the chain automatically. Store backups on S3, GCS, or Azure Blob for offsite durability.
