# How to Use the MDS Autoscaler Module in Ceph Manager

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, MDS, CephFS, Autoscaling

Description: Learn how to use the Ceph Manager MDS Autoscaler module to automatically adjust the number of active MDS daemons based on file system workload.

---

The MDS Autoscaler module in Ceph Manager automatically adjusts the `max_mds` setting for CephFS file systems based on actual workload. This eliminates the need for manual tuning and ensures the file system scales MDS daemons up under load and releases them when demand decreases.

## Enabling the MDS Autoscaler

Enable the module:

```bash
ceph mgr module enable mds_autoscaler
```

Verify it is running:

```bash
ceph mgr module ls | grep mds_autoscaler
```

## How It Works

The autoscaler monitors MDS workload metrics including:

- CPU utilization of active MDS daemons
- Number of clients connected per MDS
- Memory pressure on MDS daemons

When load exceeds a threshold, it increments `max_mds` to bring additional standby daemons into service.

## Checking Current MDS Configuration

Before enabling autoscaling, review the current MDS setup:

```bash
ceph fs status
```

```
cephfs - 3 clients
========
RANK  STATE      MDS         ACTIVITY     DNS    INOS   DIRS   CAPS
 0    active     mds.a(104)  Reqs:  50/s  5.00k  6.00k  3.00k  6.00k

STANDBY-REPLAY
STANDBY
mds.b(103)  sf: 0 (laggy: no)
```

## Setting Autoscaler Bounds

Configure minimum and maximum MDS counts on the file system:

```bash
ceph fs set cephfs min_mds 1
ceph fs set cephfs max_mds 4
```

The autoscaler respects these bounds when making scaling decisions.

## Viewing Autoscaler Activity

Check the manager logs for autoscaler decisions:

```bash
ceph log last 20 | grep mds_autoscaler
```

Or watch the MDS rank count change over time:

```bash
watch -n 5 "ceph fs status cephfs"
```

## Manual Override

To temporarily pin the number of active MDS ranks regardless of the autoscaler:

```bash
ceph fs set cephfs max_mds 2
```

The autoscaler will not scale above this value.

## Rook Integration

In a Rook-managed cluster, set MDS scaling parameters in the `CephFilesystem` custom resource:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephFilesystem
metadata:
  name: cephfs
  namespace: rook-ceph
spec:
  metadataServer:
    activeCount: 1
    activeStandby: true
```

The `activeCount` field controls the number of active MDS daemons. The autoscaler can still increase this dynamically beyond the initial value.

## Summary

The Ceph Manager MDS Autoscaler module removes the burden of manually tuning `max_mds` by automatically scaling active MDS daemons based on real-time workload metrics. Set the minimum and maximum bounds using `ceph fs set`, and the autoscaler handles dynamic adjustment within those constraints, ensuring CephFS performance adapts to changing workloads.
