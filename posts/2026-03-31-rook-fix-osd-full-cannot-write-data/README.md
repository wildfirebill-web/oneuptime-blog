# How to Fix 'osd full' and Cannot Write Data in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Troubleshooting, OSD, Full, Capacity, Storage

Description: Resolve the Ceph 'osd full' condition that blocks writes by freeing space, adding OSDs, or adjusting fullness thresholds.

---

## Introduction

When Ceph OSDs reach their fullness ratio (default 95%), the cluster blocks all write operations to prevent data corruption. This causes immediate I/O failures for all clients. This guide shows how to recover from the "osd full" condition.

## Identifying the Full Condition

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph health detail
```

Example:

```text
HEALTH_ERR 1 full osd(s); 1 nearfull osd(s)
osd.2 is full at 96%
```

Check disk usage:

```bash
ceph df
ceph osd df
```

## Immediate Recovery - Raise the Full Ratio (Temporary)

This buys time to add capacity or delete data:

```bash
# Raise thresholds temporarily
ceph osd set-full-ratio 0.97
ceph osd set-backfillfull-ratio 0.95
ceph osd set-nearfull-ratio 0.93
```

After raising the ratio, verify writes resume:

```bash
# Test write
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rados -p replicapool put test-object /etc/hostname
```

## Freeing Space - Delete Unnecessary Data

Identify large objects or buckets:

```bash
# List pools and usage
rados df

# List large objects in a pool
rados -p replicapool ls | while read obj; do
  size=$(rados -p replicapool stat $obj | awk '{print $4}')
  echo "$size $obj"
done | sort -n | tail -20
```

Delete unused snapshots that may be consuming space:

```bash
rbd snap ls replicapool/myvol
rbd snap purge replicapool/myvol
```

## Adding More OSDs

The permanent fix is expanding storage capacity:

```yaml
# In CephCluster spec, add a new node or device
storage:
  nodes:
  - name: "worker-4"
    devices:
    - name: "sdb"
    - name: "sdc"
```

Apply and wait for new OSDs:

```bash
kubectl apply -f cephcluster.yaml
kubectl -n rook-ceph get pods -l app=rook-ceph-osd --watch
```

## Rebalancing After Adding OSDs

After adding OSDs, Ceph rebalances data automatically. Monitor progress:

```bash
ceph status
watch "ceph df"
```

Once usage drops below the nearfull threshold, restore normal ratios:

```bash
ceph osd set-full-ratio 0.95
ceph osd set-backfillfull-ratio 0.90
ceph osd set-nearfull-ratio 0.85
```

## Setting Up Alerts Before Reaching Full

Configure alerting at 75% to prevent future incidents:

```bash
# Via Prometheus/Alertmanager, check for:
# ceph_osd_stat_bytes_used / ceph_osd_stat_bytes > 0.75
```

In Rook's CephCluster, ensure Prometheus is enabled:

```yaml
monitoring:
  enabled: true
```

## Summary

The "osd full" condition blocks writes at 95% OSD capacity. Immediate recovery involves temporarily raising the fullness ratio to restore write access, followed by either deleting unnecessary data or adding new OSDs for permanent relief. Setting alerting thresholds at 75% capacity prevents the cluster from reaching the full condition unexpectedly.
