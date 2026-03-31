# How to Fix "Cannot open file" Errors in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, File System, Storage, Troubleshooting, Operations

Description: Diagnose and fix ClickHouse "Cannot open file" errors caused by permission issues, missing data files, and filesystem problems.

---

## Understanding the Error

ClickHouse raises this error when it cannot access a file it needs for reading or writing:

```text
DB::Exception: Cannot open file /var/lib/clickhouse/data/analytics/events/20240115_1_5_2/data.bin. (CANNOT_OPEN_FILE)
```

Common causes include:
- Insufficient file system permissions
- Data directory moved or deleted
- Disk full preventing file creation
- Too many open files (ulimit exceeded)
- Corrupted filesystem

## Step 1 - Check File System Permissions

```bash
# Check ownership and permissions of the data directory
ls -la /var/lib/clickhouse/data/analytics/events/

# ClickHouse runs as the 'clickhouse' user - files must be owned by it
stat /var/lib/clickhouse/data/analytics/events/20240115_1_5_2/data.bin

# Fix permissions if needed
chown -R clickhouse:clickhouse /var/lib/clickhouse/
chmod -R 750 /var/lib/clickhouse/data/
```

## Step 2 - Verify the File Exists

```bash
# Check if the file is present
ls -la /var/lib/clickhouse/data/analytics/events/20240115_1_5_2/

# Expected files in a MergeTree part:
# checksums.txt, columns.txt, count.txt, data.bin, data.mrk3, primary.idx
```

If files are missing, the part may be incomplete or corrupt (see the Part is broken recovery guide).

## Step 3 - Check Open File Limits

ClickHouse opens many files simultaneously. If `ulimit` is too low, it cannot open more:

```bash
# Check current limits for the clickhouse process
cat /proc/$(pgrep -f clickhouse-server)/limits | grep "Max open files"

# Check system-wide
ulimit -n

# Check ClickHouse's configured limit
cat /etc/security/limits.d/clickhouse.conf
```

If the limit is low, increase it:

```bash
# /etc/security/limits.d/clickhouse.conf
clickhouse soft nofile 262144
clickhouse hard nofile 262144
```

Also set it in the ClickHouse config:

```xml
<!-- config.xml -->
<max_open_files>262144</max_open_files>
```

## Step 4 - Check Disk Space

A full disk prevents creating new files:

```bash
# Check disk usage
df -h /var/lib/clickhouse/

# If full, see the disk space troubleshooting guide
# Quick fix: remove old parts
clickhouse-client --query "
ALTER TABLE analytics.events DROP PARTITION '2023-01'
"
```

## Step 5 - Check for Filesystem Errors

```bash
# Look for I/O errors in the kernel log
dmesg | grep -i "error\|i/o\|filesystem" | tail -50

# Check filesystem status
mount | grep clickhouse
tune2fs -l /dev/nvme0n1p1 | grep "Filesystem state"

# Run fsck if the filesystem is unmounted
fsck -n /dev/nvme0n1p1
```

## Fix - Restore Missing Data Files

If files are genuinely missing (not a permissions issue), restore from backup or replication:

```bash
# For replicated tables - detach the incomplete part
clickhouse-client --query "
ALTER TABLE analytics.events DETACH PART '20240115_1_5_2'
"

# ZooKeeper/Keeper will schedule the part to be re-fetched
# Monitor recovery
clickhouse-client --query "
SELECT type, new_part_name, source_replica
FROM system.replication_queue
WHERE table = 'events'
"
```

## Fix - Repair /tmp Directory

ClickHouse writes temporary files to its tmp directory. If it cannot write there:

```bash
# Check tmp directory
ls -la /var/lib/clickhouse/tmp/

# Fix ownership
chown clickhouse:clickhouse /var/lib/clickhouse/tmp/
chmod 750 /var/lib/clickhouse/tmp/
```

## Monitoring for File Errors

```bash
# Monitor ClickHouse error log for file errors
tail -f /var/log/clickhouse-server/clickhouse-server.err.log | grep -i "cannot open\|permission\|no such file"

# Check open file count in real time
watch -n 5 "ls /proc/\$(pgrep -f clickhouse-server)/fd | wc -l"
```

## Summary

ClickHouse "Cannot open file" errors typically stem from permission misconfigurations, missing data files, exceeded file descriptor limits, or full disks. Start by checking file ownership with `ls -la` and fixing permissions with `chown`. If files are genuinely missing, use replication recovery or backup restoration. Always set `ulimit` to at least 262144 for the clickhouse user to prevent file descriptor exhaustion on large deployments.
