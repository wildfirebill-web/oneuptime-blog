# How to Fix 'pgs degraded' After OSD Failure in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Troubleshooting, PG, OSD, Degraded, Recovery

Description: Recover degraded placement groups after an OSD failure in Ceph by replacing the failed OSD or allowing automatic replication recovery.

---

## Introduction

When an OSD fails, Ceph marks affected PGs as degraded - meaning those PGs have fewer replicas than desired. The cluster remains operational (writes still succeed) but data durability is reduced until recovery completes. This guide covers how to handle OSD failures and ensure full recovery.

## Identifying Degraded PGs

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph health detail
```

Example:

```text
HEALTH_WARN 24 pgs degraded; 1 osd down
```

Check detailed PG status:

```bash
ceph pg stat
ceph status
```

## Step 1 - Identify the Failed OSD

```bash
ceph osd tree | grep -E "down|out"
ceph osd stat
```

Find which node the failed OSD was on:

```bash
ceph osd find osd.5
```

## Step 2 - Check if the OSD Node is Accessible

```bash
kubectl get nodes
kubectl -n rook-ceph get pods -l app=rook-ceph-osd -o wide | grep worker-3
```

If the node is temporarily unavailable (rebooting):

```bash
# Wait and watch for recovery to begin automatically
watch "ceph -s"
```

Ceph waits `mon_osd_down_out_interval` (default 10 minutes) before marking the OSD `out` and starting recovery.

## Step 3 - Manually Start Recovery

To speed up recovery if the OSD is confirmed permanently failed:

```bash
# Mark the OSD out immediately
ceph osd out osd.5

# Verify recovery has started
ceph -s | grep recovery
```

Monitor recovery progress:

```bash
watch "ceph -s | grep -E 'degraded|recovering|pgmap'"
```

## Step 4 - Remove the Failed OSD Permanently

After data has been replicated to other OSDs:

```bash
# Stop and remove the OSD
kubectl -n rook-ceph delete pod rook-ceph-osd-5-<pod-id>

# Purge the OSD from the cluster
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash
ceph osd purge osd.5 --yes-i-really-mean-it
```

## Step 5 - Add a Replacement OSD

Replace the disk or add a new node:

```yaml
# In CephCluster spec, add the replacement device
storage:
  nodes:
  - name: "worker-3"
    devices:
    - name: "sdc"  # Replacement disk
```

```bash
kubectl apply -f cephcluster.yaml
kubectl -n rook-ceph get pods -l app=rook-ceph-osd --watch
```

## Monitoring Full Recovery

```bash
# Watch until all PGs return to active+clean
watch "ceph pg stat"

# Confirm OSD tree is healthy
ceph osd tree
```

Full recovery time depends on how much data needs to be re-replicated. Expect 1-4 hours per TB on typical spinning disks.

## Summary

Degraded PGs after an OSD failure indicate reduced data redundancy. Ceph automatically begins recovery by re-replicating data to remaining OSDs after `mon_osd_down_out_interval`. Speeding recovery involves manually marking the OSD out, monitoring progress, and replacing the hardware once data has been fully re-replicated to maintain the desired replication factor.
