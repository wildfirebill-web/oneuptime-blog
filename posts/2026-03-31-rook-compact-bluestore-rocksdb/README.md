# How to Compact BlueStore RocksDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, BlueStore, RocksDB, Compaction, OSD, Performance, Maintenance

Description: Learn when and how to compact BlueStore's RocksDB metadata database to reclaim space and improve OSD metadata performance.

---

## Overview

BlueStore uses an embedded RocksDB database to store all OSD metadata including object names, xattrs, and omap data. RocksDB is a log-structured merge-tree (LSM) database that accumulates multiple levels of sorted files (SST files). Over time, without regular compaction, RocksDB performance degrades as it must search through more SST levels. This guide covers manual and automatic RocksDB compaction for BlueStore OSDs.

## Why RocksDB Compaction Matters

RocksDB works by:
1. Writing new data to an in-memory memtable
2. Flushing memtables to Level 0 (L0) SST files
3. Compacting L0 files into larger Level 1, 2, N files

When compaction lags, L0 file count grows and read latency increases as RocksDB must search more files:

```bash
# Check L0 file count
ceph daemon osd.0 perf dump | python3 -c "
import sys, json
data = json.load(sys.stdin)
rdb = data.get('rocksdb', {})
print('L0 files:', rdb.get('l0_file_count_limit_slowdowns', 0))
print('L0 stops:', rdb.get('l0_file_count_limit_stops', 0))
print('Compaction queue:', rdb.get('estimate_pending_compaction_bytes', 0))
"
```

## Triggering Manual Compaction

### Using the Admin Socket

The simplest way to trigger RocksDB compaction is via the admin socket:

```bash
# Compact all RocksDB on OSD 0
ceph daemon osd.0 compact

# This triggers a full RocksDB compaction - may take minutes to hours for large OSDs
```

Monitor progress by watching disk I/O:

```bash
iostat -x /dev/sdb 5
```

### Using ceph-kvstore-tool

For offline compaction (OSD must be stopped):

```bash
# Stop the OSD
systemctl stop ceph-osd@0

# Compact using the kvstore tool
ceph-kvstore-tool bluestore-kv /var/lib/ceph/osd/ceph-0 compact

# Restart the OSD
systemctl start ceph-osd@0
```

## Compacting RocksDB via Rook

For Rook-managed OSDs, exec into the OSD pod:

```bash
# Find the OSD pod
kubectl -n rook-ceph get pods -l app=rook-ceph-osd,ceph-osd-id=0

# Exec and trigger compaction
kubectl -n rook-ceph exec rook-ceph-osd-0-xxxx -- \
  ceph daemon /run/ceph/ceph-osd.0.asok compact
```

## Automatic Compaction Settings

Configure automatic compaction tuning via Ceph config:

```bash
# Increase compaction thread count for faster background compaction
ceph config set osd rocksdb_max_background_compactions 2
ceph config set osd rocksdb_max_background_jobs 4

# Increase L0 compaction trigger
ceph config set osd rocksdb_level0_file_num_compaction_trigger 4

# L0 slow down trigger
ceph config set osd rocksdb_level0_slowdown_writes_trigger 20

# L0 stop writes trigger
ceph config set osd rocksdb_level0_stop_writes_trigger 36
```

## RocksDB Compaction Style

BlueStore supports two RocksDB compaction styles:

```bash
# View current compaction style
ceph config show osd.0 bluestore_rocksdb_options | grep compaction

# Level-based (default) - better read performance
ceph config set osd bluestore_rocksdb_options "compaction_style=level,write_buffer_size=268435456"

# Universal - better write performance for write-heavy workloads
ceph config set osd bluestore_rocksdb_options "compaction_style=universal"
```

## Checking Compaction Effectiveness

Before compaction:

```bash
ceph daemon osd.0 perf dump | python3 -c "
import sys, json
data = json.load(sys.stdin)
rdb = data.get('rocksdb', {})
print('SST file size before:', rdb.get('live_sst_files_size', 0))
print('Pending compaction bytes:', rdb.get('estimate_pending_compaction_bytes', 0))
"
```

After compaction:

```bash
ceph daemon osd.0 compact

sleep 60  # Wait for compaction to complete

ceph daemon osd.0 perf dump | python3 -c "
import sys, json
data = json.load(sys.stdin)
rdb = data.get('rocksdb', {})
print('SST file size after:', rdb.get('live_sst_files_size', 0))
print('Pending compaction bytes:', rdb.get('estimate_pending_compaction_bytes', 0))
"
```

## Scheduling Regular Compaction

For large clusters, schedule weekly compaction during off-peak hours:

```bash
# cron job to compact all OSDs on Sunday at 2am
echo "0 2 * * 0 root for OSD in \$(ceph osd ls); do ceph daemon osd.\$OSD compact; sleep 60; done" \
  > /etc/cron.d/ceph-rocksdb-compact
```

## Summary

BlueStore RocksDB compaction prevents metadata performance degradation caused by L0 file accumulation in the LSM tree. Manual compaction via `ceph daemon osd.X compact` is the simplest remediation for affected OSDs, while tuning `rocksdb_max_background_compactions` ensures background compaction keeps pace with write load. Monitor L0 file counts and pending compaction bytes as early warning signals that compaction is falling behind.
