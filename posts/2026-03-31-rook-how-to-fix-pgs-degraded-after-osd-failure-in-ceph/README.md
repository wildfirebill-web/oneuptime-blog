# How to Fix 'pgs degraded' After OSD Failure in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, OSD, Recovery, Troubleshooting

Description: Recover degraded placement groups in Ceph after an OSD failure by restoring OSDs, replacing failed disks, and monitoring the recovery process.

---

## What "pgs degraded" Means

When one or more OSDs go down in a Ceph cluster, the placement groups (PGs) that were using those OSDs lose replicas. A PG is "degraded" when it has fewer replicas than configured, but the data is still accessible (at least `min_size` copies exist). Degraded PGs are at risk - if another OSD fails before recovery completes, data may become unavailable.

## Step 1 - Assess the Damage

Check cluster status:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph status
```

Output:

```text
health: HEALTH_WARN
    Degraded data redundancy: 1234/5000 objects degraded (24.68%)
    1 osds down
    1 host (osd.2) is down
```

Get detailed OSD information:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd tree
```

```text
-1 root default
-3     host worker-3
 2       osd.2 down weight 1.0
```

## Step 2 - Check if the OSD Pod Is Running

```bash
kubectl -n rook-ceph get pods -l app=rook-ceph-osd
```

Look for pods that are not in `Running` state. Check logs:

```bash
kubectl -n rook-ceph logs <failed-osd-pod>
```

Common causes:
- Disk hardware failure
- Filesystem corruption on the OSD volume
- Node went offline

## Step 3 - Attempt to Restart the OSD

If the disk is healthy but the OSD crashed:

```bash
kubectl -n rook-ceph delete pod <failed-osd-pod>
```

Kubernetes will restart the pod. Monitor recovery:

```bash
kubectl -n rook-ceph get pods -l app=rook-ceph-osd -w
```

If the OSD comes back up, Ceph will begin recovery automatically.

## Step 4 - Set the noout Flag if Recovery Will Take Time

If the OSD is expected to be down for a while (hardware repair, etc.):

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd set noout
```

This prevents Ceph from declaring the OSD permanently failed and starting rebalancing, which would cause unnecessary data movement.

## Step 5 - Mark OSD Out and Replace if the Disk Failed

If the disk has physically failed:

```bash
# Mark the OSD as out to begin data migration away from it
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd out osd.2
```

Monitor data migration:

```bash
watch "kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd df"
```

After data migration completes, remove the OSD from Ceph:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd purge osd.2 --yes-i-really-mean-it
```

## Step 6 - Replace the Failed Disk

After physically replacing the disk, clean it and let Rook discover it:

```bash
# On the node, wipe the new disk
sudo wipefs -a /dev/sdb
sudo dd if=/dev/zero of=/dev/sdb bs=1M count=100
```

Rook's operator will discover the new disk and create a new OSD automatically if `useAllDevices: true`.

If using specific device configuration, force a reconcile by touching the `CephCluster`:

```bash
kubectl -n rook-ceph annotate cephcluster rook-ceph rook.io/force-reconcile="$(date +%s)" --overwrite
```

## Step 7 - Monitor Recovery

Watch recovery progress:

```bash
watch -n5 "kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph status"
```

Recovery status will show:

```text
recovery: 500 MiB/s, 250 objects/s
pgs: 800/5000 objects degraded (16.0%)
     128 active+recovering
     896 active+clean
```

## Step 8 - Tune Recovery Speed

Balance recovery speed vs. client performance:

```bash
# Speed up recovery (impacts client I/O)
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph tell osd.* injectargs '--osd-max-backfills=4'
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph tell osd.* injectargs '--osd-recovery-max-active=4'

# Slow down recovery (protect client I/O)
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph tell osd.* injectargs '--osd-max-backfills=1'
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph tell osd.* injectargs '--osd-recovery-op-priority=1'
```

## Step 9 - Unset noout After Recovery

If you set `noout`, remember to unset it:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd unset noout
```

Then verify the cluster is healthy:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph health
```

Expected: `HEALTH_OK`

## Summary

Degraded PGs after an OSD failure are Ceph's normal response to reduced data redundancy. Recovery is automatic once the OSD is restored. For permanent disk failures, mark the OSD `out` to migrate data, purge it from the cluster, physically replace the disk, and allow Rook to provision a new OSD. Monitor recovery with `ceph status` and tune recovery I/O parameters to balance recovery speed against client workload performance.
