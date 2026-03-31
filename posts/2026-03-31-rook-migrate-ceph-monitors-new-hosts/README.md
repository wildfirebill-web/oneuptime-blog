# How to Migrate Ceph Monitors to New Hosts

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Monitor, Migration, Cluster Management

Description: Learn how to safely move Ceph monitor daemons to new hosts by adding new mons and removing old ones while maintaining quorum throughout.

---

Ceph monitors maintain the cluster map and require a healthy quorum to operate. Migrating monitors to new hosts must be done carefully - always maintaining quorum throughout the process.

## Understanding Monitor Quorum

A cluster with N monitors requires (N/2)+1 to form quorum:
- 3 mons: need 2 for quorum
- 5 mons: need 3 for quorum

Never remove a mon before adding its replacement when running with the minimum recommended count.

## Checking Current Monitor Status

```bash
ceph mon stat
ceph mon dump
ceph quorum_status --format json-pretty | python3 -m json.tool
```

## Step 1 - Add New Monitor on New Host

Using cephadm:

```bash
ceph orch daemon add mon new-host-1:192.168.1.10
```

Wait for the new mon to join quorum:

```bash
ceph mon stat
# Should show 4 mons if you started with 3
```

## Step 2 - Verify New Monitor is in Quorum

```bash
ceph quorum_status --format json-pretty | \
  python3 -m json.tool | grep -A5 quorum_names
```

The new monitor name should appear in the quorum list.

## Step 3 - Remove the Old Monitor

```bash
ceph mon remove old-mon-a
```

Verify quorum is maintained:

```bash
ceph mon stat
ceph quorum_status --format json-pretty
```

## Migrating Monitors in Rook

In Rook, monitor placement is controlled by the CephCluster spec. Update the mon placement:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  mon:
    count: 3
    allowMultiplePerNode: false
  placement:
    mon:
      nodeAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
          nodeSelectorTerms:
          - matchExpressions:
            - key: ceph-mon
              operator: In
              values:
              - "true"
```

Label new nodes and remove labels from old nodes:

```bash
# Add label to new nodes
kubectl label node new-node-1 ceph-mon=true
kubectl label node new-node-2 ceph-mon=true
kubectl label node new-node-3 ceph-mon=true

# Remove label from old nodes (forces Rook to migrate mons)
kubectl label node old-node-1 ceph-mon-
kubectl label node old-node-2 ceph-mon-
kubectl label node old-node-3 ceph-mon-
```

Rook will automatically failover mons to the newly labeled nodes.

## Force Mon Failover in Rook

If a mon is stuck on an old node:

```bash
kubectl -n rook-ceph delete pod rook-ceph-mon-a-xxxx
```

Rook reschedules it according to the placement policy.

## Verifying Successful Migration

```bash
ceph mon dump
kubectl -n rook-ceph get pods -l app=rook-ceph-mon -o wide
```

Confirm all mons are running on the new hosts.

## Summary

Migrating Ceph monitors to new hosts requires a strict add-then-remove sequence to maintain quorum throughout. In Rook, node affinity labels direct monitor placement, and pod deletion triggers rescheduling on the correct nodes. Always verify quorum health between each step to prevent accidental quorum loss during migration.
