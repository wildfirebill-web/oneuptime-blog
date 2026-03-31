# How to Set Up Scheduled Snapshots in CephFS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, CephFS, Snapshot, Automation

Description: Learn how to configure scheduled automatic snapshots in CephFS using the snapshot scheduling module for automated point-in-time backup in Rook-Ceph.

---

## Overview

CephFS includes a built-in snapshot scheduling module that automates periodic snapshot creation and retention. This module eliminates the need for external cron jobs or scripts to manage snapshot lifecycles. Snapshots can be scheduled at regular intervals (hourly, daily, weekly) with configurable retention counts, making it easy to implement a rolling backup policy for CephFS subvolumes.

## Enable the Snapshot Scheduling Module

The snapshot scheduling feature is implemented as a Ceph Manager module:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph mgr module enable snap_schedule
```

Verify it is active:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph mgr module ls | grep snap_schedule
```

## Add a Snapshot Schedule

Schedule daily snapshots at midnight for the root of a filesystem:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph fs snap-schedule add / 1d --fs cephfs
```

Add an hourly snapshot schedule for a specific directory:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph fs snap-schedule add /myapp 1h --fs cephfs
```

Supported schedule intervals:

```text
1m   - Every minute (for testing)
1h   - Hourly
1d   - Daily
1w   - Weekly
```

## Set Retention Policy

Configure how many snapshots to retain per schedule:

```bash
# Keep the last 24 hourly snapshots
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph fs snap-schedule retention add /myapp h 24 --fs cephfs

# Keep the last 7 daily snapshots
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph fs snap-schedule retention add /myapp d 7 --fs cephfs

# Keep the last 4 weekly snapshots
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph fs snap-schedule retention add /myapp w 4 --fs cephfs
```

## List Current Schedules

View all configured schedules:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph fs snap-schedule list / --fs cephfs --recursive
```

## View Snapshot Schedule Status

Check the status of scheduled snapshots including last execution time:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph fs snap-schedule status /myapp --fs cephfs
```

## Remove a Schedule

```bash
# Remove hourly schedule
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph fs snap-schedule remove /myapp 1h --fs cephfs

# Remove all schedules for a path
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph fs snap-schedule remove /myapp --fs cephfs
```

## Verify Snapshots Are Being Created

After waiting for the first scheduled execution, verify snapshots exist:

```bash
ls /mnt/cephfs/myapp/.snap/
```

Snapshot names follow the pattern: `scheduled-YYYY-MM-DD-HH:MM:SS`

## Configure via Rook CephFilesystem CRD

Rook also supports snapshot scheduling through the `snapshotSchedules` field in the CephFilesystem CRD:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephFilesystem
metadata:
  name: cephfs
  namespace: rook-ceph
spec:
  metadataPool:
    replicated:
      size: 3
  dataPools:
    - name: data0
      replicated:
        size: 3
  metadataServer:
    activeCount: 1
    activeStandby: true
```

## Summary

The CephFS snapshot scheduling module provides automated, policy-driven snapshot management with configurable intervals and retention counts. By enabling the `snap_schedule` manager module and adding schedules with `ceph fs snap-schedule add` and retention rules with `ceph fs snap-schedule retention add`, you can implement a comprehensive rolling snapshot policy for your Rook-Ceph CephFS filesystems without external automation tools.
