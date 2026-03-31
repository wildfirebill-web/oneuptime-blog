# How to Start, Stop, and Restart a Ceph Cluster

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Operation, Cluster, Maintenance

Description: Learn how to safely start, stop, and restart a Ceph cluster managed by Rook for maintenance, upgrades, and emergency procedures.

---

## Overview

In a Rook-managed Ceph cluster, daemons run as Kubernetes pods. Starting and stopping the cluster means scaling Kubernetes deployments and DaemonSets, or using Rook's built-in cluster pause functionality.

Ceph daemon ordering matters: monitors must be up before OSDs, and OSDs must be running before MDS or RGW daemons can serve clients.

## Pausing the Rook Operator

The safest way to stop Rook from making changes is to pause the operator:

```bash
# Scale down the Rook operator to prevent it from reconciling
kubectl -n rook-ceph scale deployment rook-ceph-operator --replicas=0
```

## Stopping All Ceph Daemons

To stop all Ceph daemons for cluster maintenance:

```bash
# 1. Set the cluster to noout to prevent CRUSH rebalancing during shutdown
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd set noout

# 2. Set norebalance
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd set norebalance

# 3. Stop RGW/MDS first
kubectl -n rook-ceph scale deployment -l app=rook-ceph-rgw --replicas=0
kubectl -n rook-ceph scale deployment -l app=rook-ceph-mds --replicas=0

# 4. Stop OSDs
kubectl -n rook-ceph scale deployment -l app=rook-ceph-osd --replicas=0

# 5. Stop monitors last
kubectl -n rook-ceph scale deployment -l app=rook-ceph-mon --replicas=0
```

## Restarting the Cluster

Restart in reverse order - monitors first, then OSDs, then services:

```bash
# 1. Start monitors
kubectl -n rook-ceph scale deployment -l app=rook-ceph-mon --replicas=1

# Wait for monitors to form quorum
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph mon stat

# 2. Start OSDs
kubectl -n rook-ceph scale deployment -l app=rook-ceph-osd --replicas=1

# 3. Clear maintenance flags
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd unset noout

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd unset norebalance

# 4. Start RGW and MDS
kubectl -n rook-ceph scale deployment -l app=rook-ceph-rgw --replicas=1
kubectl -n rook-ceph scale deployment -l app=rook-ceph-mds --replicas=2

# 5. Restart the Rook operator
kubectl -n rook-ceph scale deployment rook-ceph-operator --replicas=1
```

## Restarting a Single Daemon

To restart a specific OSD without touching the rest of the cluster:

```bash
kubectl -n rook-ceph rollout restart deployment rook-ceph-osd-2
```

## Verifying Cluster Health After Restart

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph -s
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph health detail
```

## Summary

In Rook, start and stop Ceph daemons by scaling Kubernetes deployments. Always set `noout` before stopping OSDs to prevent unnecessary data rebalancing. Follow the correct order: monitors first on startup, monitors last on shutdown. Use `ceph -s` to verify cluster health at each step.
