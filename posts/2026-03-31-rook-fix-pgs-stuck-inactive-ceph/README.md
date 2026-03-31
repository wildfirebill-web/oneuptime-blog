# How to Fix "pgs are stuck inactive" in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Troubleshooting, PG, Placement Group, Recovery

Description: Diagnose and fix inactive placement groups in Ceph by identifying blocked OSDs, resolving CRUSH map issues, and restoring cluster health.

---

## Introduction

Inactive placement groups (PGs) mean Ceph cannot service I/O for data in those PGs. This causes client-facing read/write failures. PGs become inactive when all OSDs responsible for them are down or unavailable. This guide walks through diagnosing and resolving stuck inactive PGs.

## Identifying the Problem

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph health detail
```

Look for output like:

```
HEALTH_ERR 5 pgs are stuck inactive for more than 300 seconds
5 pgs are stuck inactive
```

Get the specific PG IDs:

```bash
ceph pg dump | grep -E "^[0-9]+\.[0-9]+" | awk '$9 != "active+clean" {print $1, $9}'
```

## Step 1 - Check Which OSDs Are Involved

```bash
ceph pg map 1.5
```

Output:

```
osdmap e123 pg 1.5 (1.5) -> up [2,4,7] acting [2,4,7]
```

Check if those OSDs are up:

```bash
ceph osd tree
ceph osd status
```

## Step 2 - Check OSD Pod Status

```bash
kubectl -n rook-ceph get pods -l app=rook-ceph-osd
kubectl -n rook-ceph describe pod rook-ceph-osd-2-<pod-id>
```

If pods are crashing, check logs:

```bash
kubectl -n rook-ceph logs rook-ceph-osd-2-<pod-id> --previous
```

## Step 3 - Force Mark OSDs Up (Temporary)

If OSDs are temporarily unavailable during maintenance:

```bash
ceph osd up 2
ceph osd in 2
```

## Step 4 - Check CRUSH Map

An incorrect CRUSH map can cause PGs to have no valid acting set:

```bash
ceph osd crush tree
ceph osd crush rule dump
```

If the crush rule requires a minimum number of hosts but fewer exist:

```bash
# Check pool crush rule
ceph osd pool get replicapool crush_rule

# Modify the CRUSH rule to allow fewer failure domains
ceph osd crush rule create-simple \
  my-rule default osd
ceph osd pool set replicapool crush_rule my-rule
```

## Step 5 - Reduce Minimum Size Temporarily

If not enough OSDs exist to satisfy `min_size`:

```bash
ceph osd pool get replicapool min_size
# If min_size is 2 but only 1 OSD is available:
ceph osd pool set replicapool min_size 1
```

This allows the cluster to recover before restoring the minimum size.

## Step 6 - Force PG Recovery

```bash
ceph pg repair 1.5
ceph pg force-recovery 1.5
```

Monitor recovery progress:

```bash
watch ceph status
```

## Summary

Stuck inactive PGs result from all responsible OSDs being unavailable, CRUSH map misconfiguration, or pool size requirements exceeding available OSDs. The resolution path involves identifying affected OSDs, addressing why they are down, and temporarily reducing pool constraints if necessary to allow the cluster to become active before restoring normal redundancy settings.
