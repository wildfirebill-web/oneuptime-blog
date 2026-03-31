# How to Fix 'health HEALTH_ERR' After Cluster Deployment

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Troubleshooting, Health, Error, Cluster

Description: Diagnose and resolve HEALTH_ERR status in a newly deployed Rook-Ceph cluster by reading health details and addressing root causes.

---

## Introduction

Seeing `HEALTH_ERR` after deploying a Rook-Ceph cluster is alarming but usually fixable. The error state covers many possible underlying issues. This guide shows how to systematically read health details and address the most common causes.

## Reading the Health Details

Never act on `HEALTH_ERR` alone - always read the full detail:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph health detail
```

Example output:

```text
HEALTH_ERR 1 filesystem is damaged; 2 pgs are degraded; mon quorum is not live
```

Or check the full status:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph status
```

## Common HEALTH_ERR Causes After Deployment

### 1. Not Enough OSDs for Pool Replication

A pool requiring 3 replicas but only 2 OSDs are available causes PG errors:

```bash
ceph health detail
# OUTPUT: pg 1.0 is degraded, acting [0,1] (need 3)
```

Fix by reducing pool size during development:

```bash
ceph osd pool set replicapool size 2
ceph osd pool set replicapool min_size 1
```

Or add more OSD nodes to match the desired replica count.

### 2. Clock Skew Between Monitors

Monitors require clocks within 0.05 seconds of each other:

```bash
ceph health detail
# OUTPUT: clock skew detected on mon.b
```

Fix:

```bash
# On each node
systemctl restart chronyd
# Or
ntpdate -u pool.ntp.org
```

### 3. Monitors Not in Quorum

Check monitor status:

```bash
ceph mon stat
ceph mon dump
```

If a monitor pod is crashing:

```bash
kubectl -n rook-ceph logs rook-ceph-mon-a-<pod-id> | tail -30
```

Restart the monitor pod:

```bash
kubectl -n rook-ceph delete pod rook-ceph-mon-a-<pod-id>
```

### 4. OSD Pods Failing to Start

```bash
kubectl -n rook-ceph get pods | grep osd
kubectl -n rook-ceph logs rook-ceph-osd-0-<pod-id> | grep -i error
```

Common causes: disk permissions, missing LVM packages on the node.

Install required packages on the node if needed:

```bash
apt-get install -y lvm2 ceph-common
```

## Using the Rook Toolbox for Diagnostics

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash

# Inside toolbox:
ceph health detail
ceph osd tree
ceph df
ceph pg stat
```

## Automated Health Check Script

```bash
#!/bin/bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash -c "
  echo '=== Cluster Status ==='
  ceph status
  echo '=== OSD Tree ==='
  ceph osd tree
  echo '=== PG Summary ==='
  ceph pg stat
  echo '=== Recent Events ==='
  ceph log last 20
"
```

## Summary

HEALTH_ERR after cluster deployment usually stems from insufficient OSDs for the configured pool replication size, clock skew between monitor nodes, or OSD pods failing to start due to disk or dependency issues. Reading `ceph health detail` provides the specific error code needed to target the correct fix, and most issues are resolved by matching the replication factor to available OSDs or correcting node-level clock synchronization.
