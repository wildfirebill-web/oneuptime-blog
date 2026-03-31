# How to Use cephfs-journal-tool for Recovery

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, CephFS, Journal, Recovery

Description: Learn how to use cephfs-journal-tool to inspect and recover a corrupt CephFS MDS journal in Rook-Ceph environments.

---

## What Is the CephFS MDS Journal

The CephFS MDS journal is a write-ahead log stored in the metadata pool. All metadata changes are written to the journal before being committed to the backing metadata store. If the MDS crashes while the journal is partially written, the journal must be replayed or reset to restore consistency.

The `cephfs-journal-tool` utility lets you inspect journal contents, export events, and reset the journal when it is corrupt or unrecoverable.

## Step 1 - Take the Filesystem Offline

Before running `cephfs-journal-tool`, ensure the MDS is fully stopped:

```bash
kubectl exec -it deploy/rook-ceph-tools -n rook-ceph -- \
  ceph fs fail myfs

kubectl patch cephfilesystem myfs -n rook-ceph \
  --type merge -p '{"spec":{"metadataServer":{"activeCount":0}}}'

kubectl exec -it deploy/rook-ceph-tools -n rook-ceph -- \
  ceph fs dump | grep state
```

Wait until no MDS is active.

## Step 2 - Inspect the Journal

Use `cephfs-journal-tool` to inspect the journal header and events:

```bash
kubectl exec -it deploy/rook-ceph-tools -n rook-ceph -- \
  cephfs-journal-tool --rank myfs:0 journal inspect
```

Sample output indicating a corrupt journal:

```text
Journal magic 0xc00l appears valid
Journal events: 15234
Header: expire_pos=0x1a3f4b, write_pos=0x1a4000
ERROR: commit pos 0x1a3f00 < expire pos 0x1a3f4b
```

## Step 3 - Export Journal Events to File

Before any destructive operations, export the current journal events for inspection:

```bash
kubectl exec -it deploy/rook-ceph-tools -n rook-ceph -- \
  cephfs-journal-tool --rank myfs:0 event get list
```

Export specific events to a local file via the exec session:

```bash
kubectl exec -it deploy/rook-ceph-tools -n rook-ceph -- \
  cephfs-journal-tool --rank myfs:0 event get json --output /tmp/journal-events.json
```

## Step 4 - Recover Dentries from the Journal

Attempt to recover directory entries that have been journaled but not committed to the backing store:

```bash
kubectl exec -it deploy/rook-ceph-tools -n rook-ceph -- \
  cephfs-journal-tool --rank myfs:0 event recover_dentries list
```

Apply the recovery:

```bash
kubectl exec -it deploy/rook-ceph-tools -n rook-ceph -- \
  cephfs-journal-tool --rank myfs:0 event recover_dentries apply
```

## Step 5 - Reset the Journal

If the journal is corrupt beyond recovery, reset it. This discards uncommitted metadata:

```bash
kubectl exec -it deploy/rook-ceph-tools -n rook-ceph -- \
  cephfs-journal-tool --rank myfs:0 journal reset
```

After reset, also run `cephfs-table-tool` to reset the MDS tables:

```bash
kubectl exec -it deploy/rook-ceph-tools -n rook-ceph -- \
  cephfs-table-tool all reset session

kubectl exec -it deploy/rook-ceph-tools -n rook-ceph -- \
  cephfs-table-tool all reset snap

kubectl exec -it deploy/rook-ceph-tools -n rook-ceph -- \
  cephfs-table-tool all reset inode
```

## Step 6 - Bring the Filesystem Back Online

Re-enable the filesystem and start MDS daemons:

```bash
kubectl exec -it deploy/rook-ceph-tools -n rook-ceph -- \
  ceph fs set myfs joinable true

kubectl patch cephfilesystem myfs -n rook-ceph \
  --type merge -p '{"spec":{"metadataServer":{"activeCount":1}}}'
```

Monitor MDS startup for any remaining errors:

```bash
kubectl logs -n rook-ceph deploy/rook-ceph-mds-myfs-a --follow
```

## Summary

`cephfs-journal-tool` is a critical recovery tool when the CephFS MDS journal becomes corrupt or unreadable. The recovery sequence is: take the filesystem offline, inspect the journal, attempt dentry recovery, and reset the journal if necessary. Always export journal events before making destructive changes. After journal reset, restart the MDS and run a full filesystem scrub to validate metadata consistency.
