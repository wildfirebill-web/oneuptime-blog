# How to Configure CephFS Snapshot Mirroring

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, Storage, Kubernetes, Disaster Recovery

Description: Learn how to configure CephFS snapshot mirroring to asynchronously replicate filesystem snapshots to a remote Ceph cluster for disaster recovery.

---

## What Is CephFS Snapshot Mirroring

CephFS snapshot mirroring is an asynchronous replication feature that copies snapshots from one CephFS filesystem to another, typically on a remote cluster. It was introduced in Ceph Pacific (16.x) and provides a disaster recovery mechanism for CephFS workloads without requiring synchronous writes.

The mirroring daemon (`cephfs-mirror`) monitors configured directories for snapshots and syncs them to the peer cluster. This is directory-level replication, not full filesystem replication, which means you can selectively mirror specific directories.

## Prerequisites

Both the source and destination clusters must be running Ceph Pacific or later. You need:
- Admin credentials for both clusters
- Network connectivity between the clusters
- CephFS filesystems configured on both clusters
- The `cephfs-mirror` daemon deployed on the source cluster

## Enabling the Mirroring Module

On the source cluster, enable the mirroring module:

```bash
ceph mgr module enable mirroring
```

Enable mirroring on the CephFS filesystem:

```bash
ceph fs snapshot mirror enable myfs
```

Verify mirroring is active:

```bash
ceph fs snapshot mirror show myfs
```

## Deploying the cephfs-mirror Daemon

Deploy the mirror daemon using cephadm on the source cluster:

```bash
ceph orch apply cephfs-mirror
```

Check that the daemon is running:

```bash
ceph orch ls | grep cephfs-mirror
ceph orch ps | grep cephfs-mirror
```

## Creating a Peer Bootstrap Token

On the destination cluster, generate a bootstrap token that the source will use to authenticate:

```bash
ceph fs snapshot mirror peer_bootstrap create destfs client.mirror_peer /
```

This command outputs a token string. Copy it securely to the source cluster.

## Adding the Remote Peer on the Source

On the source cluster, import the bootstrap token to register the peer:

```bash
ceph fs snapshot mirror peer_bootstrap import myfs \
  eyJmc2lkIjoiY2ZhNGE3MTMtNWNkMy00MjA0LWI3...
```

Verify the peer is registered:

```bash
ceph fs snapshot mirror peer list myfs
```

## Configuring Directories to Mirror

Enable mirroring for specific directories within the filesystem:

```bash
ceph fs snapshot mirror add myfs /volumes/data
ceph fs snapshot mirror add myfs /volumes/backups
```

List the directories being mirrored:

```bash
ceph fs snapshot mirror dirmap myfs
```

## Creating Snapshots for Mirroring

The mirror daemon syncs snapshots, not live data. Create snapshots under the mirrored directory:

```bash
mkdir -p /mnt/cephfs/volumes/data/.snap/daily-2026-03-31
```

Or use the CephFS snapshot command if using kernel mount:

```bash
mount -t ceph :/ /mnt/cephfs -o name=admin
mkdir /mnt/cephfs/volumes/data/.snap/snap1
```

The mirror daemon will detect new snapshots and begin syncing to the peer cluster.

## Monitoring Mirror Status

Check synchronization status:

```bash
ceph fs snapshot mirror status myfs
```

For per-directory status:

```bash
ceph fs snapshot mirror dirmap myfs /volumes/data
```

Look for `snap_count`, `last_synced_snap`, and `last_sync_duration` in the output to confirm mirroring is progressing.

## Configuring in Rook

In a Rook-managed cluster, enable snapshot mirroring in the `CephFilesystem` spec:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephFilesystem
metadata:
  name: myfs
  namespace: rook-ceph
spec:
  metadataPool:
    replicated:
      size: 3
  dataPools:
    - name: data0
      replicated:
        size: 3
  mirroring:
    enabled: true
    snapshotSchedules:
      - path: /
        interval: 24h
```

Apply and check the generated bootstrap secret for the peer:

```bash
kubectl apply -f filesystem.yaml
kubectl -n rook-ceph get secret rook-ceph-filesystem-myfs-peer -o yaml
```

## Summary

CephFS snapshot mirroring provides asynchronous disaster recovery by syncing filesystem snapshots to a remote Ceph cluster. Enable the mirroring module on the source filesystem, deploy the `cephfs-mirror` daemon, register the remote peer using a bootstrap token, and configure which directories to mirror. Monitor sync status with `ceph fs snapshot mirror status`. Rook simplifies this with the `mirroring` section in the `CephFilesystem` spec.
