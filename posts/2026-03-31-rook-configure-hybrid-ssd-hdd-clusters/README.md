# How to Configure Ceph for Hybrid SSD/HDD Clusters

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, SSD, HDD, Hybrid, Tiering, Configuration

Description: Configure Ceph with mixed SSD and HDD drives to use SSDs as BlueStore WAL/DB devices and create tiered storage pools for different performance requirements.

---

## Hybrid Architecture Overview

A hybrid SSD/HDD Ceph cluster uses SSDs to accelerate HDD performance by placing BlueStore WAL and RocksDB metadata on SSDs while bulk data remains on HDDs. This gives you close-to-SSD metadata performance at HDD data costs.

## Configure OSD WAL/DB on SSD

In Rook, specify the WAL and DB device per OSD:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  storage:
    nodes:
      - name: "worker-01"
        devices:
          - name: "sdb"   # HDD data device
            config:
              metadataDevice: "nvme0n1"   # SSD for DB + WAL
          - name: "sdc"   # HDD data device
            config:
              metadataDevice: "nvme0n1"
          - name: "sdd"
            config:
              metadataDevice: "nvme1n1"
```

## SSD Sizing for WAL/DB

```bash
# BlueStore RocksDB (DB) size: ~1-4% of HDD OSD size
# HDD OSD: 4 TB -> DB: ~40-160 GB

# Rule of thumb:
# 1 NVMe can back the DB for 5-6 x 4 TB HDDs
# 1 NVMe can back the DB for 3-4 x 8 TB HDDs

# Monitor actual DB usage
ceph daemon osd.0 perf dump | grep bluefs
```

## Create Separate Device Class Pools

```bash
# CRUSH rules for each tier
ceph osd crush rule create-replicated hdd-rule default host hdd
ceph osd crush rule create-replicated ssd-rule default host ssd
```

## Pool Configuration for Tiered Workloads

```yaml
# HDD pool for bulk/cold data
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: hdd-bulk
  namespace: rook-ceph
spec:
  failureDomain: host
  replicated:
    size: 3
  parameters:
    crush_rule: hdd-rule
    compression_mode: aggressive
---
# SSD pool for hot/latency-sensitive data
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: ssd-hot
  namespace: rook-ceph
spec:
  failureDomain: host
  replicated:
    size: 3
  parameters:
    crush_rule: ssd-rule
```

## Monitor BlueFS SSD Usage

```bash
# Check how much SSD is being used by BlueFS
ceph daemon osd.0 perf dump | python3 -c "
import json, sys
d = json.load(sys.stdin)
bf = d.get('bluefs', {})
for k, v in bf.items():
    if 'bytes' in k.lower():
        print(f'{k}: {v/(1024**3):.2f} GB')
"

# Verify DB device assignment
ceph osd metadata osd.0 | grep -E "device|bluefs"
```

## Tune BlueStore for Hybrid Setups

```bash
# For HDDs with SSD WAL/DB - larger RocksDB WAL in SSD
ceph config set osd bluestore_rocksdb_options \
  "compression=kNoCompression,max_write_buffer_number=4,min_write_buffer_number_to_merge=1"

# BlueFS will automatically prefer writing to the faster device
ceph config set osd bluefs_shared_alloc_size 65536
```

## Summary

Hybrid SSD/HDD Ceph clusters deliver a compelling price-performance balance: HDDs provide low-cost bulk capacity while SSDs accelerate BlueStore metadata and WAL operations. By configuring Rook to assign SSD metadata devices per HDD OSD, and creating separate CRUSH pools for hot and cold workloads, you get tiered storage that satisfies both performance and cost requirements.
