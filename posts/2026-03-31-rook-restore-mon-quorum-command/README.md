# How to Restore Mon Quorum Using the restore-quorum Command in Rook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Monitor, Quorum, Recovery

Description: Step-by-step guide to using Rook's restore-quorum command to recover a Ceph cluster when monitor quorum cannot be re-established automatically.

---

When a Ceph cluster loses monitor quorum and normal recovery methods fail, Rook provides a `restore-quorum` operation that rebuilds the quorum from a single surviving monitor. This is a last-resort procedure - it modifies the monitor map and should only be used when you have verified that normal quorum cannot be restored.

## When to Use restore-quorum

Use this procedure when:
- At least one monitor is still running and healthy
- Two or more monitors are permanently lost (node failure, data corruption)
- Normal mon pod restarts have failed to restore quorum
- `ceph status` hangs or times out

Do not use this if monitors are only temporarily unavailable due to a network issue or node reboot.

## Prerequisites

Identify the surviving monitor. Check which mon pods are running:

```bash
kubectl -n rook-ceph get pods -l app=rook-ceph-mon -o wide
```

```text
NAME                  READY   STATUS      RESTARTS   AGE   NODE
rook-ceph-mon-a-xxx   2/2     Running     0          10d   node-1
rook-ceph-mon-b-xxx   0/2     OOMKilled   5          2h    node-2
rook-ceph-mon-c-xxx   0/2     Error       3          2h    node-3
```

In this example, `mon-a` on `node-1` is the surviving monitor.

## Initiating restore-quorum

Rook exposes the restore-quorum operation through the CephCluster CR. Edit the cluster resource to specify which monitor should be used as the quorum basis:

```bash
kubectl -n rook-ceph edit cephcluster rook-ceph
```

Add the following annotation to trigger the restore-quorum operation:

```yaml
metadata:
  annotations:
    ceph.rook.io/restore-mon-quorum: "a"
```

The value `"a"` corresponds to the monitor ID of the surviving monitor. Save and exit. The Rook operator will detect this annotation and begin the restore process.

## Monitoring the Restore Process

Watch the operator logs to follow progress:

```bash
kubectl -n rook-ceph logs -l app=rook-ceph-operator -f
```

You should see log lines indicating the restore-quorum operation is proceeding. The operator will:
1. Scale down all other monitor deployments
2. Patch the monitor map to contain only the surviving monitor
3. Restart the surviving monitor with the new map
4. Gradually bring additional monitors back online

Check the operator events:

```bash
kubectl -n rook-ceph get events --sort-by='.lastTimestamp' | tail -20
```

## Verifying Recovery

Once the operation completes, verify the cluster is back in quorum:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph status
```

```text
cluster:
  id:     xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
  health: HEALTH_WARN
          1 mon is down (out of 3)
          Degraded data redundancy...

services:
  mon: 1 daemons, quorum a (age 5m)
  mgr: a(active, since 3m)
  osd: 6 osds: 6 up, 6 in
```

The cluster will initially show HEALTH_WARN because only one monitor is active. The Rook operator will then automatically add new monitors to restore the full three-monitor configuration.

## Rebuilding the Monitor Ensemble

After restoring quorum with a single monitor, allow Rook to rebuild the full set:

```bash
kubectl -n rook-ceph get pods -l app=rook-ceph-mon -w
```

Rook will create new mon deployments on available nodes. This process takes several minutes. Once three monitors are healthy and in quorum, remove the restore annotation:

```bash
kubectl -n rook-ceph annotate cephcluster rook-ceph ceph.rook.io/restore-mon-quorum-
```

## Post-Recovery Checks

After full quorum is restored, verify OSD health and data recovery:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph health detail
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd stat
```

Any PGs that were degraded due to the quorum loss should begin recovering automatically.

## Summary

The `restore-quorum` operation in Rook is triggered by annotating the CephCluster CR with the name of the surviving monitor. The Rook operator handles the low-level monitor map manipulation needed to re-establish quorum from a single node. After using this procedure, allow Rook to rebuild the full three-monitor ensemble before removing the annotation and resuming normal operations.
