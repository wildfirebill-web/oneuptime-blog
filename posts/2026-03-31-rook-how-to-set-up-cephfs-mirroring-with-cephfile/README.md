# How to Set Up CephFS Mirroring with CephFilesystemMirror CRD

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Cephfs, Mirroring, Kubernetes, Disaster Recovery

Description: Learn how to configure CephFS mirroring using the CephFilesystemMirror CRD in Rook for cross-cluster filesystem replication and disaster recovery.

---

## Overview

CephFS mirroring replicates filesystem snapshots from one Ceph cluster to another, enabling disaster recovery and geo-redundancy. Rook manages the mirroring daemon through the `CephFilesystemMirror` CRD and enables snapshot-based replication between clusters.

## Architecture

CephFS mirroring works by:
1. A mirroring daemon on the source cluster monitors snapshots
2. Changed data is replicated to the target cluster
3. Snapshots on the source are synchronized to corresponding directories on the target

You need to deploy the CephFilesystemMirror on the source cluster and configure peer credentials from the target cluster.

## Step 1 - Deploy the CephFilesystemMirror

Create the mirror daemon on the source cluster:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephFilesystemMirror
metadata:
  name: my-fs-mirror
  namespace: rook-ceph
spec:
  resources:
    requests:
      cpu: "100m"
      memory: "512Mi"
    limits:
      memory: "1Gi"
  priorityClassName: system-cluster-critical
```

Apply the manifest:

```bash
kubectl apply -f filesystem-mirror.yaml
```

Verify the mirror daemon is running:

```bash
kubectl -n rook-ceph get pod -l app=rook-ceph-fs-mirror
```

## Step 2 - Enable Mirroring on the Filesystem

Update the CephFilesystem to enable mirroring:

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
    - name: replicated
      replicated:
        size: 3
  mirroring:
    enabled: true
    peers:
      secretNames:
        - fs-peer-secret
  metadataServer:
    activeCount: 1
    activeStandby: true
```

## Step 3 - Bootstrap Peer Credentials from Target Cluster

On the target cluster, bootstrap a token for the source to use:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph fs snapshot mirror peer_bootstrap create myfs client.mirror /
```

This outputs a base64-encoded token. Create a secret on the source cluster:

```bash
kubectl -n rook-ceph create secret generic fs-peer-secret \
  --from-literal=token=<base64-token-from-target>
```

## Step 4 - Configure Snapshot Schedules

Enable snapshot scheduling on the source filesystem directory:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph fs snapshot schedule add myfs / 1h
```

List configured schedules:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph fs snapshot schedule list myfs /
```

## Step 5 - Verify Mirroring Status

Check the mirroring status for your filesystem:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph fs snapshot mirror show myfs
```

Expected output:

```text
{
    "peers": [
        {
            "uuid": "...",
            "remote": {
                "client_name": "client.mirror",
                "cluster_name": "target-cluster",
                "fs_name": "myfs"
            },
            "stats": {
                "failure_count": 0,
                "recovery_count": 0
            }
        }
    ],
    "snap_dirs": {
        "/": {
            "mirror_enabled": true
        }
    }
}
```

## Monitoring Mirroring Health

Check replication lag and sync status:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph fs snapshot mirror daemon status
```

## Summary

CephFS mirroring via the CephFilesystemMirror CRD in Rook deploys a dedicated mirror daemon that replicates filesystem snapshots to a remote Ceph cluster. The process involves deploying the mirror CRD, enabling mirroring on the filesystem, exchanging peer bootstrap tokens between clusters, and configuring snapshot schedules. This setup provides automated cross-cluster data replication for disaster recovery scenarios.
