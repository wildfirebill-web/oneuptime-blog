# How to Handle Out-of-Disk-Space Situations in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Disk, Storage, Recovery, Emergency

Description: Learn immediate steps to recover MongoDB from an out-of-disk-space event and long-term strategies to prevent disk exhaustion in production.

---

## What Happens When MongoDB Runs Out of Disk

When the disk containing MongoDB's `dbPath` fills up, writes fail with `ENOSPC` (no space left on device). MongoDB may also fail to write journal files, which can corrupt in-progress write operations. In a replica set, the affected node falls behind oplog replication and eventually goes stale.

Acting quickly and in the right order prevents data corruption and minimizes downtime.

## Immediate Response: Free Disk Space Fast

### 1. Identify what is consuming space

```bash
# Check overall disk usage
df -h /var/lib/mongodb

# Find largest files and directories
du -sh /var/lib/mongodb/* | sort -rh | head -20

# Check MongoDB log files (often large)
du -sh /var/log/mongodb/
```

### 2. Rotate and compress log files

```bash
# Signal mongod to rotate logs
mongosh --eval "db.adminCommand({ logRotate: 1 })"

# Compress old logs
gzip /var/log/mongodb/mongod.log.20*

# Or truncate in-use log if rotation fails
> /var/log/mongodb/mongod.log
```

### 3. Delete temp files

```bash
# Remove MongoDB temp files from incomplete operations
rm -rf /var/lib/mongodb/_tmp/*

# Remove core dump files if present
find /var/lib/mongodb -name "core.*" -delete
```

## Freeing Space from MongoDB Data

If log cleanup is not enough, reclaim space from data:

```javascript
// Drop collections that are no longer needed
db.temp_exports.drop();

// Remove old documents from large collections
db.audit_logs.deleteMany({
  createdAt: { $lt: new Date(Date.now() - 30 * 24 * 60 * 60 * 1000) }
});

// Run compact after deletion (during maintenance window)
db.runCommand({ compact: "audit_logs" });
```

## Expanding Disk Without Downtime

On cloud providers, expanding the volume is the fastest fix:

```bash
# AWS - resize an EBS volume
aws ec2 modify-volume --volume-id vol-xxxxxxxx --size 500

# After resize, grow the filesystem (no reboot needed on Linux)
sudo growpart /dev/xvdf 1
sudo resize2fs /dev/xvdf1
```

For bare-metal servers, add a new disk and use LVM to extend the volume group:

```bash
sudo pvcreate /dev/sdb
sudo vgextend mongodb_vg /dev/sdb
sudo lvextend -l +100%FREE /dev/mongodb_vg/data_lv
sudo resize2fs /dev/mongodb_vg/data_lv
```

## Preventing Future Disk Exhaustion

Set up alerting before the disk fills:

```bash
# Simple cron-based alert at 80% usage
*/5 * * * * df /var/lib/mongodb | awk 'NR==2 {gsub("%",""); if ($5 > 80) print "ALERT: MongoDB disk " $5 "% full"}' | mail -s "MongoDB disk alert" ops@example.com
```

Configure MongoDB Atlas or Prometheus with an alert rule:

```yaml
# Prometheus alert rule
- alert: MongoDBDiskUsageHigh
  expr: (node_filesystem_size_bytes{mountpoint="/var/lib/mongodb"} - node_filesystem_avail_bytes{mountpoint="/var/lib/mongodb"}) / node_filesystem_size_bytes{mountpoint="/var/lib/mongodb"} > 0.8
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "MongoDB disk usage above 80%"
```

## Replica Set Recovery After Disk Full

After freeing space, the node may need to resync:

```javascript
// Check replication status
rs.status();

// If the node is stale (oplog gap too large), perform initial sync
// Remove all data files and let the node resync from primary
// WARNING: only do this on a secondary
```

```bash
sudo systemctl stop mongod
sudo rm -rf /var/lib/mongodb/*
sudo systemctl start mongod
# Mongod will now perform initial sync from the primary
```

## Summary

When MongoDB runs out of disk, immediately rotate logs, delete temp files, and remove dispensable data. Expand the volume using cloud tools or LVM without restarting MongoDB. After space is freed, verify replica set health with `rs.status()` and resync stale secondaries if needed. Long-term, implement disk usage alerts at 70-80% and use TTL indexes plus scheduled archiving to keep data volumes bounded.
