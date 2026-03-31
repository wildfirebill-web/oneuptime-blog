# How to Back Up ClickHouse to Google Cloud Storage

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Backup, Google Cloud Storage, GCS, Disaster Recovery

Description: Learn how to configure ClickHouse BACKUP commands to write directly to Google Cloud Storage for durable, offsite backups.

---

ClickHouse's native BACKUP/RESTORE feature supports writing backups directly to Google Cloud Storage (GCS). This provides durable, offsite backup storage without requiring intermediate local disk space.

## Prerequisites

You need a GCS bucket and service account credentials. Create a service account with Storage Object Admin role and download the JSON key file.

## Configuring GCS Storage in ClickHouse

Add GCS as a storage endpoint in `config.d/gcs-backup.xml`:

```xml
<clickhouse>
    <storage_configuration>
        <disks>
            <gcs_backup>
                <type>object_storage</type>
                <object_storage_type>gcs</object_storage_type>
                <metadata_type>plain_rewritable</metadata_type>
                <bucket>my-clickhouse-backups</bucket>
                <path>backups/</path>
            </gcs_backup>
        </disks>
    </storage_configuration>
    <backups>
        <allowed_disk>gcs_backup</allowed_disk>
        <allowed_path>/</allowed_path>
    </backups>
</clickhouse>
```

## Setting Up Authentication

ClickHouse uses Application Default Credentials for GCS. Set the credentials path:

```bash
export GOOGLE_APPLICATION_CREDENTIALS=/etc/clickhouse-server/gcs-key.json
```

Or mount the key file in your ClickHouse container and reference it in the environment.

## Creating a Full Backup to GCS

Run a full backup of a database:

```sql
BACKUP DATABASE my_database
TO Disk('gcs_backup', 'my_database_backup_2026-03-31/')
SETTINGS async = true;
```

Back up a specific table:

```sql
BACKUP TABLE my_database.events
TO Disk('gcs_backup', 'events_backup_2026-03-31/')
SETTINGS async = true;
```

## Monitoring Backup Progress

Check the status of an async backup:

```sql
SELECT
    id,
    status,
    start_time,
    end_time,
    num_files,
    total_size,
    exception
FROM system.backups
ORDER BY start_time DESC
LIMIT 5;
```

## Creating Incremental Backups

Use the `base_backup` option to create incremental backups that only store changed data:

```sql
-- Full backup on Monday
BACKUP DATABASE my_database
TO Disk('gcs_backup', 'full_backup_2026-03-31/');

-- Incremental backup on Tuesday
BACKUP DATABASE my_database
TO Disk('gcs_backup', 'incremental_2026-04-01/')
SETTINGS base_backup = Disk('gcs_backup', 'full_backup_2026-03-31/');
```

## Restoring from GCS

Restore a backup from GCS to a new database:

```sql
RESTORE DATABASE my_database AS my_database_restored
FROM Disk('gcs_backup', 'my_database_backup_2026-03-31/');
```

Restore a specific table:

```sql
RESTORE TABLE my_database.events AS my_database.events_restored
FROM Disk('gcs_backup', 'events_backup_2026-03-31/');
```

## Automating with Shell Script

Automate daily backups with a shell script and cron:

```bash
#!/bin/bash
DATE=$(date +%Y-%m-%d)
clickhouse-client --query "BACKUP DATABASE production TO Disk('gcs_backup', 'prod_backup_${DATE}/')"
echo "Backup completed: prod_backup_${DATE}"
```

Schedule it with cron:

```bash
0 2 * * * /usr/local/bin/clickhouse-backup-gcs.sh >> /var/log/clickhouse-backup.log 2>&1
```

## Summary

Backing up ClickHouse to GCS requires configuring a GCS disk in `storage_configuration`, setting up Application Default Credentials, and using the native BACKUP command. Incremental backups with `base_backup` reduce storage costs for daily backup schedules. Monitor backup status via `system.backups` and test restores regularly.
