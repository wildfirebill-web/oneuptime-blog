# How to Plan Disk Layout for Ceph (Data, WAL, DB)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Disk, Layout, WAL, BlueStore, Hardware, OSD

Description: Design the optimal disk layout for Ceph BlueStore OSDs by understanding the roles of data, WAL, and RocksDB partition placement and their performance impact.

---

## Overview

Ceph BlueStore uses three logical storage areas per OSD: data, WAL (write-ahead log), and RocksDB metadata DB. Placing WAL and DB on fast NVMe while data goes on HDDs dramatically improves write performance and metadata operations.

## BlueStore Storage Roles

- **Data**: The actual object data stored by Ceph. Can be on HDD for cost-effective storage.
- **WAL (Write-Ahead Log)**: Small sequential writes for crash consistency. Very I/O intensive, benefits from fast SSD.
- **DB (RocksDB metadata)**: Key-value metadata store. Random read/write intensive, benefits greatly from NVMe.

## Option 1: All-on-Same-Device (Simple)

All three areas on the same disk:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
spec:
  storage:
    useAllNodes: true
    useAllDevices: false
    devices:
      - name: sda
      - name: sdb
```

Best for: all-SSD or all-NVMe clusters where cost isn't a concern.

## Option 2: HDD Data + NVMe WAL/DB (Recommended for HDD)

Place WAL and DB on NVMe, data on HDD:

```yaml
spec:
  storage:
    useAllNodes: true
    useAllDevices: false
    devices:
      - name: sda  # HDD
        config:
          metadataDevice: nvme0n1
      - name: sdb  # HDD
        config:
          metadataDevice: nvme0n1
      - name: sdc  # HDD
        config:
          metadataDevice: nvme0n1
```

## Step 1 - Calculate NVMe Partition Sizes

```bash
# Per OSD on an HDD:
# WAL: 512 MB to 2 GB (usually 1 GB is plenty)
# DB: 64 GB to 4% of OSD data size (whichever is larger)

# For 12 TB HDD:
# DB = max(64 GB, 12 TB * 0.04) = max(64 GB, 491 GB)
# Practical DB size: 64-100 GB per HDD OSD

# For 12 HDDs sharing one NVMe (1 TB):
# WAL per OSD: 1 GB x 12 = 12 GB
# DB per OSD: 64 GB x 12 = 768 GB
# Total NVMe needed: 780 GB -> 1 TB NVMe handles 12 HDDs
```

## Step 2 - Configure Rook with Explicit Partitions

```yaml
spec:
  storage:
    nodes:
      - name: osd-node-1
        devices:
          - name: sda  # 12 TB HDD
            config:
              metadataDevice: nvme0n1
          - name: sdb  # 12 TB HDD
            config:
              metadataDevice: nvme0n1
        config:
          osdsPerDevice: "1"
          walSizeMB: "1024"
          dbSizeMB: "65536"
```

## Step 3 - Verify OSD Placement

```bash
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  ceph osd metadata 0 | grep -E "backend|wal|db|bluestore"
```

Output example:

```json
{
  "osd_objectstore": "bluestore",
  "bluestore_bdev_devices": "sda",
  "bluestore_bdev_type": "hdd",
  "bluefs_dedicated_db": "1",
  "bluefs_db_dev": "nvme0n1p2"
}
```

## Step 4 - Monitor WAL/DB Utilization

```bash
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  ceph daemon osd.0 bluestore bluefs stats
```

Key metrics to watch:
- `db_used_bytes`: If approaching partition size, DB needs expansion
- `wal_bytes`: Should stay well below WAL partition size

## Step 5 - DB Overflow Warning

When DB fills up, data spills to the HDD which hurts performance:

```bash
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  ceph daemon osd.0 perf dump | grep bluefs
```

Look for `db_used_bytes` approaching `db_total_bytes`.

## Sizing Reference Table

```
HDD Size | Recommended DB Size | WAL Size
---------|---------------------|----------
4 TB     | 32 GB               | 1 GB
8 TB     | 64 GB               | 1 GB
12 TB    | 96 GB               | 1 GB
16 TB    | 128 GB              | 1 GB
```

## Summary

Separating WAL and DB onto NVMe devices is the single most impactful hardware optimization for HDD-based Ceph clusters, improving write latency by 5-10x compared to all-on-HDD configurations. Rook's `metadataDevice` configuration makes this setup straightforward. Right-size your DB partitions at roughly 1% of the HDD capacity (minimum 32 GB) to avoid the performance penalty of DB overflow onto the HDD.
