# How to Configure BlueStore DB Device Settings

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, BlueStore, DB Device, RocksDB, OSD, Performance, NVMe

Description: Configure BlueStore's RocksDB metadata database on a separate SSD or NVMe device to improve metadata-intensive workload performance.

---

## Overview

BlueStore stores all OSD metadata (object names, xattrs, omap data) in an embedded RocksDB database. By default, this database lives on the same block device as the OSD data. Moving the RocksDB DB to a faster SSD or NVMe drive improves metadata-intensive operations, particularly for workloads with many small objects or heavy omap usage (such as RGW bucket indexes).

## What Is Stored on the DB Device

The BlueStore DB device (also called the "block.db") stores:

- Object namespace and name mappings
- Extended attributes (xattrs)
- Omap data (key-value data for objects)
- RocksDB internal metadata and SST files

## Performance Impact

Operations that benefit most from a fast DB device:

- RGW bucket listing (reads omap indexes)
- CephFS directory operations
- Snapshot creation (metadata-heavy)
- Metadata pool workloads

## Configuring a Separate DB Device

### With cephadm

```bash
# Add OSD with separate DB on SSD and WAL on NVMe
ceph orch daemon add osd \
  myhost:/dev/sdb:data \
  /dev/ssd0:/dev/nvme0n1:block_db:block_wal
```

Or specify separately:

```bash
ceph orch daemon add osd \
  myhost:/dev/sdb:data:/dev/ssd0:db
```

### With ceph-volume

```bash
ceph-volume lvm prepare \
  --data /dev/sdb \
  --block.db /dev/ssd0
```

### With Rook-Ceph CephCluster

```yaml
# rook-cluster-db.yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  storage:
    storageClassDeviceSets:
      - name: hdd-set
        count: 6
        volumeClaimTemplates:
          - metadata:
              name: data
            spec:
              storageClassName: local-hdd
              resources:
                requests:
                  storage: 4Ti
          - metadata:
              name: metadata
            spec:
              storageClassName: local-ssd
              resources:
                requests:
                  storage: 50Gi
```

## DB Device Sizing Guidelines

The DB device needs to be large enough to hold RocksDB without spillover to the main device. If the DB device fills up, BlueStore falls back to the main block device (which is slower).

```bash
# Rule of thumb: 1-2% of OSD data size
# For a 1TB OSD: 10-20 GB DB device

# Check current RocksDB size
du -sh /var/lib/ceph/osd/ceph-0/block.db
```

Recommended sizing per OSD:

| OSD Size | Minimum DB Size | Recommended DB Size |
|---|---|---|
| 1 TB | 5 GB | 10 GB |
| 4 TB | 20 GB | 40 GB |
| 10 TB | 50 GB | 100 GB |

## Verifying DB Device Usage

```bash
# Check DB device symlink
ls -la /var/lib/ceph/osd/ceph-0/block.db

# Verify it points to the SSD
stat $(readlink -f /var/lib/ceph/osd/ceph-0/block.db) | grep Device
```

## Monitoring DB Device Metrics

```bash
# Check RocksDB compaction stats
ceph daemon osd.0 perf dump | python3 -c "
import sys, json
data = json.load(sys.stdin)
rocksdb = data.get('rocksdb', {})
print('Compactions:', rocksdb.get('compact_range_count', 0))
print('L0 files:', rocksdb.get('l0_file_count_limit_slowdowns', 0))
"
```

Monitor DB device I/O:

```bash
iostat -x /dev/ssd0 5
```

## Handling DB Device Overflow

If the DB device fills up:

```bash
# Check DB space usage
ceph daemon osd.0 perf dump | grep bluestore_db

# If overflowing, increase DB size or move RocksDB back
# Option: compact RocksDB to reduce size
ceph daemon osd.0 compact
```

## Summary

Moving BlueStore's RocksDB metadata database to a dedicated SSD or NVMe device reduces metadata operation latency, which is critical for workloads with high omap usage or many small objects. Size the DB device at 1-2% of the OSD data size to prevent DB spillover to the slower main device. Monitor RocksDB compaction rates and L0 file counts to identify when the DB device is under pressure.
