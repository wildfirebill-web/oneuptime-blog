# How to Configure OSD Settings in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, OSD, Storage, Configuration

Description: Learn how to configure Ceph OSD settings including scrubbing, recovery, and performance tuning for optimal cluster behavior.

---

## Overview of OSD Configuration

Object Storage Daemons (OSDs) are the workhorses of a Ceph cluster. They handle data storage, replication, recovery, and rebalancing. Fine-tuning OSD settings is critical for achieving the right balance between performance and data safety.

OSD settings can be applied via `ceph.conf` or dynamically using `ceph config set`. In Rook, you can inject configuration via the `CephCluster` CR's `cephConfig` field.

## Common OSD Settings

### Scrubbing

Scrubbing verifies data consistency. Light scrubbing checks object metadata; deep scrubbing also checksums the data:

```ini
[osd]
osd_scrub_begin_hour = 22
osd_scrub_end_hour = 6
osd_scrub_sleep = 0.1
osd_deep_scrub_interval = 604800
```

Setting `osd_scrub_begin_hour` and `osd_scrub_end_hour` restricts scrubbing to off-peak hours.

### Recovery and Backfill

Control how aggressively Ceph recovers data after failures:

```ini
[osd]
osd_recovery_max_active = 3
osd_recovery_op_priority = 3
osd_max_backfills = 1
osd_backfill_scan_min = 64
osd_backfill_scan_max = 512
```

Lower `osd_recovery_op_priority` gives production I/O priority over recovery.

### Applying Settings in Rook

You can set OSD config dynamically using the toolbox:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set osd osd_recovery_max_active 3
```

Or via the `CephCluster` CR:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  cephConfig:
    osd:
      osd_scrub_begin_hour: "22"
      osd_scrub_end_hour: "6"
      osd_recovery_max_active: "3"
```

## OSD Capacity Settings

### Full Ratio Thresholds

Ceph uses three thresholds to manage capacity:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set global mon_osd_full_ratio 0.95

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set global mon_osd_nearfull_ratio 0.85

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set global mon_osd_backfillfull_ratio 0.90
```

## Verifying OSD Configuration

List all current OSD configuration values:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config dump | grep osd
```

Check a specific OSD's running config:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config show osd.0
```

## Summary

OSD configuration in Ceph covers scrubbing intervals, recovery aggressiveness, and capacity thresholds. In Rook environments, apply settings either dynamically via `ceph config set` in the toolbox or declaratively through the `CephCluster` CR's `cephConfig` field. Start with conservative recovery settings in production to avoid impacting I/O performance during rebalancing.
