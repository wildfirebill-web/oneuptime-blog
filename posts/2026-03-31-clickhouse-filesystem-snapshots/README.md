# How to Use Filesystem Snapshots for ClickHouse Backups

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Backup, Filesystem Snapshot, LVM, Data Protection

Description: Learn how to use filesystem-level snapshots (LVM, ZFS, EBS) for consistent ClickHouse backups with minimal downtime and fast restore times.

---

Filesystem snapshots provide an alternative backup method for ClickHouse that can be faster than the native `BACKUP` command for very large datasets. By freezing the filesystem state, you capture a consistent point-in-time view of all ClickHouse data files.

## When to Use Filesystem Snapshots

Filesystem snapshots are ideal when:
- You need near-instant backup creation (snapshots are typically created in seconds)
- Your data volume is very large (tens of terabytes)
- You are running on cloud VMs with block storage (AWS EBS, GCP Persistent Disk)
- You use LVM or ZFS on-premises

## Preparing ClickHouse for Consistent Snapshots

Before taking a snapshot, freeze ClickHouse tables to ensure data consistency:

```sql
-- Freeze all tables to create hard links to data parts
ALTER TABLE analytics.events FREEZE;
ALTER TABLE analytics.sessions FREEZE;
```

Or freeze an entire database at once using a script:

```bash
clickhouse-client --query="
  SELECT concat('ALTER TABLE ', database, '.', name, ' FREEZE;')
  FROM system.tables
  WHERE database NOT IN ('system', 'information_schema')
  FORMAT LineAsString
" | clickhouse-client
```

Frozen parts are stored in `/var/lib/clickhouse/shadow/`.

## Taking an LVM Snapshot

```bash
# Freeze ClickHouse tables first (see above)

# Create LVM snapshot (requires enough free space in volume group)
lvcreate -L 50G -s -n clickhouse_snap /dev/vg0/clickhouse

# Thaw - ClickHouse can resume writes immediately
clickhouse-client --query="SYSTEM UNFREEZE WITH NAME 'backup-20260331'"

# Mount the snapshot
mount -o ro /dev/vg0/clickhouse_snap /mnt/clickhouse_snap

# Archive or sync to backup location
rsync -av /mnt/clickhouse_snap/ s3://my-bucket/snapshots/2026-03-31/

# Unmount and remove snapshot
umount /mnt/clickhouse_snap
lvremove -f /dev/vg0/clickhouse_snap
```

## Taking an AWS EBS Snapshot

```bash
# Get the volume ID
VOLUME_ID=$(aws ec2 describe-instances \
  --instance-ids i-0abc123def \
  --query 'Reservations[].Instances[].BlockDeviceMappings[?DeviceName==`/dev/sda1`].Ebs.VolumeId' \
  --output text)

# Freeze ClickHouse
clickhouse-client --query="ALTER TABLE analytics.events FREEZE;"

# Create EBS snapshot
aws ec2 create-snapshot \
  --volume-id "${VOLUME_ID}" \
  --description "ClickHouse backup $(date +%Y-%m-%d)" \
  --tag-specifications 'ResourceType=snapshot,Tags=[{Key=Name,Value=clickhouse-backup}]'

# Unfreeze
clickhouse-client --query="SYSTEM UNFREEZE WITH NAME 'temp';"
```

## Taking a ZFS Snapshot

```bash
# Create ZFS snapshot
zfs snapshot data/clickhouse@backup-2026-03-31

# List snapshots
zfs list -t snapshot

# Send to remote or S3
zfs send data/clickhouse@backup-2026-03-31 | gzip | \
  aws s3 cp - s3://my-bucket/zfs-snapshots/clickhouse-2026-03-31.gz

# Rollback to snapshot (destructive)
zfs rollback data/clickhouse@backup-2026-03-31
```

## Restoring from a Filesystem Snapshot

To restore from an LVM snapshot:

```bash
# Stop ClickHouse
systemctl stop clickhouse-server

# Mount snapshot read-only
mount -o ro /dev/vg0/clickhouse_snap /mnt/clickhouse_snap

# Copy data back
rsync -av /mnt/clickhouse_snap/ /var/lib/clickhouse/

# Fix permissions
chown -R clickhouse:clickhouse /var/lib/clickhouse/

# Start ClickHouse
systemctl start clickhouse-server
```

## Summary

Filesystem snapshots using LVM, ZFS, or cloud block storage (EBS, Persistent Disk) provide a fast alternative to ClickHouse native backups. Freeze tables before snapshotting to ensure consistency, create the snapshot quickly, then immediately unfreeze to resume writes. Snapshots are especially powerful for large datasets where native BACKUP is too slow.
