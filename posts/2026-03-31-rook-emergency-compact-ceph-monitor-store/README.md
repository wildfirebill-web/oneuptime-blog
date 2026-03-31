# How to Emergency Compact Ceph Monitor Store

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Monitor, Storage, Kubernetes, Troubleshooting

Description: Learn how to emergency compact the Ceph monitor store to recover from corruption or excessive growth that prevents normal monitor operation.

---

## Overview

The Ceph monitor store (based on RocksDB) can grow excessively or become corrupted, causing monitors to fail to start or perform poorly. Emergency compaction reduces the store size and resolves certain types of corruption by triggering a manual RocksDB compaction cycle.

## When to Use Emergency Compaction

You may need emergency compaction when:
- Monitors fail to start with database errors
- Monitor store size grows unexpectedly large
- Monitors experience excessive read latency during recovery
- RocksDB WAL (write-ahead log) files accumulate

Check current monitor store size before proceeding:

```bash
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph tell mon.* version
```

```bash
kubectl -n rook-ceph get pods -l app=rook-ceph-mon
```

## Step 1: Identify the Affected Monitor

Find the monitor that needs compaction and check its PVC:

```bash
kubectl -n rook-ceph get pvc | grep mon
```

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph mon stat
```

## Step 2: Scale Down the Monitor

Before compacting, scale down the affected monitor deployment:

```bash
kubectl -n rook-ceph scale deployment rook-ceph-mon-a --replicas=0
```

Verify it is down:

```bash
kubectl -n rook-ceph get pods -l ceph_daemon_type=mon
```

## Step 3: Run Compaction via a Debug Pod

Create a debug pod that mounts the monitor's PVC:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mon-compact-debug
  namespace: rook-ceph
spec:
  containers:
  - name: compact
    image: quay.io/ceph/ceph:v18
    command:
    - ceph-mon
    - --compact
    - -i
    - a
    - --mon-data
    - /var/lib/ceph/mon/ceph-a
    volumeMounts:
    - name: mon-data
      mountPath: /var/lib/ceph/mon/ceph-a
  volumes:
  - name: mon-data
    persistentVolumeClaim:
      claimName: rook-ceph-mon-a
  restartPolicy: Never
```

Apply and wait for completion:

```bash
kubectl apply -f mon-compact-debug.yaml
kubectl -n rook-ceph wait --for=condition=complete pod/mon-compact-debug --timeout=300s
kubectl -n rook-ceph logs mon-compact-debug
```

## Step 4: Restore the Monitor

After successful compaction, remove the debug pod and restore the monitor:

```bash
kubectl -n rook-ceph delete pod mon-compact-debug
kubectl -n rook-ceph scale deployment rook-ceph-mon-a --replicas=1
```

Verify recovery:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph -s
```

## Preventive Measures

Configure monitor compaction schedule in the CephCluster spec:

```yaml
spec:
  mon:
    count: 3
  cephVersion:
    image: quay.io/ceph/ceph:v18
  storage:
    config:
      mon_compact_on_start: "true"
      mon_rocksdb_options: "compaction_style=level"
```

## Summary

Emergency compaction of the Ceph monitor store is performed by scaling down the monitor, running `ceph-mon --compact` via a debug pod, and then restoring the deployment. This resolves RocksDB bloat and certain corruption scenarios that prevent normal monitor operation. Enabling `mon_compact_on_start` helps prevent excessive store growth going forward.
