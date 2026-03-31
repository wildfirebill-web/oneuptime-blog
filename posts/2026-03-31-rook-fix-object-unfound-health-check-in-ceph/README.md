# How to Fix OBJECT_UNFOUND Health Check in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, Object, Health Check, Recovery

Description: Learn how to resolve OBJECT_UNFOUND in Ceph, a critical health error indicating that objects cannot be located on any OSD, which may result in data loss.

---

## What Is OBJECT_UNFOUND?

`OBJECT_UNFOUND` is a critical Ceph health error that means one or more objects are in a state where Ceph knows they should exist (they are recorded in PG logs) but cannot find a valid copy on any available OSD. This is one of the most serious states in Ceph because it may indicate permanent data loss.

This condition occurs when all OSDs that held copies of an object fail simultaneously before replication completed.

## Checking the Warning

```bash
ceph health detail
```

Example output:

```text
[ERR] OBJECT_UNFOUND: 2 unfound objects
    pg 1.0 is active+recovery_wait+degraded, 2 unfound objects
```

Find exactly which objects are unfound:

```bash
ceph pg 1.0 list_unfound
```

## Diagnosing the Cause

Check which OSDs are down:

```bash
ceph osd tree | grep down
```

Check when the OSDs went down relative to when writes occurred:

```bash
ceph log last 50 | grep "osd\." | grep -E "down|boot"
```

## Recovery Options

### Option 1 - Bring Back Down OSDs

If the missing OSDs are only temporarily down (e.g., node rebooted), bringing them back will allow recovery:

```bash
kubectl -n rook-ceph get pods -l app=rook-ceph-osd
kubectl -n rook-ceph rollout restart deploy/rook-ceph-osd-<id>
```

Once the OSD comes back, check if objects are found:

```bash
watch ceph pg 1.0 query | grep unfound
```

### Option 2 - Mark Unfound Objects as Lost

If the OSDs are permanently gone and data cannot be recovered, you must make a decision about the unfound objects. This is a destructive action:

```bash
ceph pg 1.0 mark_unfound_lost revert
```

Or delete the objects entirely:

```bash
ceph pg 1.0 mark_unfound_lost delete
```

`revert` attempts to restore from a previous version if available. `delete` permanently removes the object references.

### Option 3 - Check for Backups

Before marking objects as lost, check if backups exist:

```bash
velero restore list
```

Or check application-level backups (database dumps, S3 backups, etc.).

## Preventing OBJECT_UNFOUND

Use adequate replication (size >= 3) and proper failure domains:

```bash
ceph osd pool set critical-pool size 3
ceph osd pool set critical-pool min_size 2
```

Ensure replicas are in different failure domains:

```bash
ceph osd crush rule create-replicated host-replicated default host
ceph osd pool set critical-pool crush_rule host-replicated
```

Enable Rook Velero integration for external backups:

```yaml
apiVersion: velero.io/v1
kind: BackupStorageLocation
metadata:
  name: ceph-backup
  namespace: velero
```

## Summary

`OBJECT_UNFOUND` is a critical error indicating objects with no available copies. First try bringing back down OSDs - if they return, data is recovered automatically. If OSDs are permanently gone, use `mark_unfound_lost` to resolve the state (this may result in data loss). Prevent this by maintaining replication >= 3, using proper failure domains, and keeping external backups for critical data.
