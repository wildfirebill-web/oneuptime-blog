# How to Recover a Ceph Cluster After Network Partition

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Disaster Recovery, Network Partition, Recovery, Storage

Description: Learn how to recover a Ceph cluster after a network partition event, restore monitor quorum, reconcile split-brain scenarios, and resume normal operations.

---

## What Happens During a Network Partition

A network partition splits the cluster into isolated groups. If the majority of monitors are in one partition, that group maintains quorum and stays active. The minority partition's monitors lose quorum and the cluster becomes read-only or unavailable from that segment.

## Identifying the Partition Impact

From the side with quorum:

```bash
ceph -s
ceph mon stat
ceph quorum_status
```

Example showing a minority side:

```
2024-03-10T14:22:01.123456+0000 mon.a (rank 0) 3 mons at
{a=192.168.1.1:6789/0,b=192.168.1.2:6789/0,c=192.168.1.3:6789/0},
election epoch 18, quorum 0 a
```

Only one of three monitors has quorum - clients connected to monitors b or c cannot write.

## Checking Monitor Connectivity

Test connectivity between monitors:

```bash
ceph ping mon.a
ceph ping mon.b
ceph ping mon.c
```

Check the monitor map:

```bash
ceph mon dump
```

## After the Partition Heals

When network connectivity is restored, monitors automatically re-establish quorum:

```bash
# Watch for quorum to reform
watch -n 2 ceph quorum_status

# Verify all monitors are in quorum
ceph mon stat
```

## Handling OSD Split-Brain

During a partition, some OSDs may have been marked out by the side that maintained quorum. After partition healing:

```bash
# Check for OSDs marked out during the partition
ceph osd tree | grep out

# Bring them back in if they are healthy
ceph osd in osd.<id>
```

## Recovery After Long-Duration Partition

For partitions that lasted long enough to trigger rebalancing:

```bash
# Check recovery state
ceph -s | grep recovering

# Set recovery throttling to reduce impact
ceph config set osd osd_recovery_max_active 2
ceph config set osd osd_recovery_sleep_hdd 0.5
```

## Verifying PG Consistency

After the partition heals, check for inconsistent PGs:

```bash
ceph pg dump | grep inconsistent
ceph health detail
```

Repair any inconsistencies:

```bash
ceph pg repair <pg-id>
```

## Rook Network Partition Recovery

In Kubernetes environments, network partitions can also affect the control plane. Check Rook operator status:

```bash
kubectl -n rook-ceph get pods -l app=rook-ceph-operator
kubectl -n rook-ceph get cephcluster
```

If the Rook operator lost connectivity and the cluster is stuck:

```bash
kubectl -n rook-ceph delete pod -l app=rook-ceph-operator
```

This forces the operator to restart and reconcile cluster state.

## Preventing Quorum Loss During Partitions

Ensure monitors are distributed across failure domains:

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
```

Place monitors in different availability zones or racks to minimize simultaneous partition impact.

## Summary

Network partition recovery in Ceph is mostly automatic once connectivity is restored - monitors re-establish quorum and OSDs resume peering. The key operator actions are checking for OSDs incorrectly marked out, verifying PG consistency, and throttling recovery to reduce I/O impact. In Rook, restarting the operator after prolonged partitions ensures proper cluster reconciliation.
