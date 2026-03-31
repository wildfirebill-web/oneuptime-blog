# How to Handle Full OSDs and mon_osd_full_ratio in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, OSD, Capacity, Full

Description: Learn how to handle full OSDs in Ceph, understand the full ratio thresholds, and recover a cluster that has stopped accepting writes.

---

## Ceph Capacity Thresholds

Ceph uses three capacity thresholds to protect data integrity:

| Setting | Default | Behavior |
| --- | --- | --- |
| `mon_osd_nearfull_ratio` | 0.85 | HEALTH_WARN: cluster is getting full |
| `mon_osd_backfillfull_ratio` | 0.90 | Stop backfill operations |
| `mon_osd_full_ratio` | 0.95 | HEALTH_ERR: stop accepting writes |

When an OSD hits the full ratio, all pools that map to it become read-only. This prevents data corruption from out-of-space errors.

## Checking Capacity

View current cluster capacity:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph df
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd df
```

Check health detail for full OSD messages:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph health detail
```

## Immediate Recovery Options

### Option 1 - Delete Unnecessary Data

Identify and delete large objects or snapshots:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rados -p mypool ls | head -20

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool ls detail
```

### Option 2 - Temporarily Raise Full Ratio

As a temporary emergency measure, raise the full ratio to regain write access:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd set-full-ratio 0.97

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd set-nearfull-ratio 0.90
```

Do not leave these values raised permanently - they exist to protect the cluster.

### Option 3 - Add New OSDs

The permanent fix is to add more storage capacity. In Rook, add OSDs by expanding the storage device list:

```yaml
spec:
  storage:
    nodes:
      - name: worker-4
        devices:
          - name: /dev/sdb
          - name: /dev/sdc
```

After adding OSDs, data will rebalance automatically.

### Option 4 - Delete Snapshots

Ceph RBD snapshots consume space. Remove old snapshots:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rbd snap ls mypool/myvolume

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rbd snap rm mypool/myvolume@old-snapshot
```

## Preventing Future Full OSD Events

Set up Prometheus alerts for the nearfull ratio. In Rook's monitoring setup, alert rules are available via the `CephCluster` monitoring CRDs.

Configure pool quotas to prevent individual pools from consuming all available space:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool set-quota mypool max_bytes 107374182400

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool get-quota mypool
```

## Resetting Full Ratios After Adding Capacity

After adding OSDs and allowing rebalancing to complete, reset thresholds to defaults:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd set-full-ratio 0.95

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd set-nearfull-ratio 0.85

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd set-backfillfull-ratio 0.90
```

## Summary

Full OSD events stop Ceph from accepting writes to protect data consistency. Respond by first understanding what is consuming space, then either deleting data, temporarily raising the full ratio, or adding new OSDs. Prevent future occurrences with pool quotas, capacity monitoring via Prometheus, and proactive storage expansion before hitting the nearfull threshold.
