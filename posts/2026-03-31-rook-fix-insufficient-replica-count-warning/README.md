# How to Fix "insufficient replica count" Warning in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Troubleshooting, Replica, Pool, Warning, Redundancy

Description: Resolve Ceph insufficient replica count warnings by checking pool size settings, OSD availability, and adjusting replication factors to match cluster capacity.

---

## Introduction

The "insufficient replica count" warning appears when a Ceph pool's `size` (desired replicas) cannot be met because not enough OSDs are available. While the cluster remains operational, data durability is reduced. This guide explains how to diagnose and fix this warning.

## Identifying the Warning

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph health detail
```

Example:

```
HEALTH_WARN Reduced data availability: 6 pgs inactive; insufficient replica count: 12 pgs
```

Check pool settings:

```bash
ceph osd pool ls detail
```

## Step 1 - Check OSD Availability

The warning usually means OSDs are down or out:

```bash
ceph osd tree
ceph osd stat
```

Look for OSDs with status `down` or `out`. If they are recently rebooted nodes:

```bash
# Mark OSDs back in
ceph osd in osd.2
ceph osd in osd.3
```

Wait for PGs to recover:

```bash
watch "ceph -s | grep pgmap"
```

## Step 2 - Check Pool Replication Settings

Verify what replication size is configured:

```bash
ceph osd pool get replicapool size
ceph osd pool get replicapool min_size
```

If `size` is 3 but only 2 OSDs are available, reduce the size:

```bash
ceph osd pool set replicapool size 2
```

For development clusters with limited nodes:

```bash
ceph osd pool set replicapool size 2
ceph osd pool set replicapool min_size 1
```

## Step 3 - Add More OSDs to Restore Full Redundancy

The proper fix for production is adding enough OSDs to meet the desired size:

```yaml
# In CephCluster spec, add more storage nodes
storage:
  nodes:
  - name: "worker-4"
    devices:
    - name: "sdb"
  - name: "worker-5"
    devices:
    - name: "sdb"
```

```bash
kubectl apply -f cephcluster.yaml
kubectl -n rook-ceph get pods -l app=rook-ceph-osd --watch
```

## Step 4 - Check CRUSH Rules

If OSDs exist but replicas are not being placed correctly, the CRUSH rule may be too restrictive:

```bash
ceph osd crush rule dump replicated_rule
```

If the rule requires placement on separate hosts but all OSDs are on the same host:

```bash
ceph osd crush rule create-simple relaxed-rule default osd
ceph osd pool set replicapool crush_rule relaxed-rule
```

## Step 5 - Verify Recovery After Fix

```bash
ceph health
ceph pg stat
```

Output should show all PGs as `active+clean`.

```bash
# Confirm pool replica count
ceph osd pool get replicapool size
ceph osd pool get replicapool min_size
```

## Summary

Insufficient replica count warnings indicate that Ceph cannot place the required number of object replicas due to insufficient active OSDs or restrictive CRUSH rules. Short-term fixes involve marking OSDs back in or reducing the pool size setting. Long-term resolution requires adding enough OSD nodes to satisfy the desired replication factor across separate failure domains.
