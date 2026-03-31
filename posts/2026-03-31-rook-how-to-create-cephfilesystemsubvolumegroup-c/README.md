# How to Create CephFilesystemSubVolumeGroup CRDs in Rook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Cephfs, Kubernetes, Storage, Subvolume

Description: Learn how to create and manage CephFilesystemSubVolumeGroup CRDs in Rook to organize CephFS subvolumes into logical groups with shared quotas.

---

## Overview

CephFS SubVolumeGroups allow you to organize subvolumes into logical groups within a filesystem. The `CephFilesystemSubVolumeGroup` CRD in Rook lets you declaratively manage these groups, which the CSI driver uses when provisioning volumes.

SubVolumeGroups provide:
- Logical isolation of volumes within a group
- Per-group quota enforcement
- Namespace-like organization within a filesystem

## Prerequisites

You need a running CephFilesystem before creating SubVolumeGroups:

```bash
kubectl -n rook-ceph get cephfilesystem myfs
```

## Creating a SubVolumeGroup

Define the SubVolumeGroup using the Rook CRD:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephFilesystemSubVolumeGroup
metadata:
  name: group1
  namespace: rook-ceph
spec:
  filesystemName: myfs
  name: group1
  pinning:
    distributed: 1
```

Apply the manifest:

```bash
kubectl apply -f subvolumegroup.yaml
```

The `pinning.distributed` field distributes subvolumes across MDS ranks for better load balancing.

## Pinning Strategies

Rook supports three pinning strategies for SubVolumeGroups:

### Distributed Pinning

Distributes subvolumes evenly across available MDS ranks:

```yaml
spec:
  filesystemName: myfs
  pinning:
    distributed: 1
```

### Export Pinning

Pins all subvolumes in the group to a specific MDS rank:

```yaml
spec:
  filesystemName: myfs
  pinning:
    export: 0
```

### Random Pinning

Assigns subvolumes randomly across MDS ranks with a probability:

```yaml
spec:
  filesystemName: myfs
  pinning:
    random: 0.5
```

## Configuring Quotas on the Group

Set a quota for all volumes within the SubVolumeGroup by specifying quota in the spec:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephFilesystemSubVolumeGroup
metadata:
  name: team-a-group
  namespace: rook-ceph
spec:
  filesystemName: myfs
  name: team-a
  quota:
    maxFiles: 100000
    maxBytes: "100Gi"
```

## Creating a StorageClass Referencing a SubVolumeGroup

Update the CephFS StorageClass to use a specific SubVolumeGroup:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-cephfs-team-a
provisioner: rook-ceph.cephfs.csi.ceph.com
parameters:
  clusterID: rook-ceph
  fsName: myfs
  pool: myfs-replicated
  subvolumeGroup: team-a
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-cephfs-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/controller-expand-secret-name: rook-csi-cephfs-provisioner
  csi.storage.k8s.io/controller-expand-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-cephfs-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
reclaimPolicy: Delete
allowVolumeExpansion: true
```

## Verifying the SubVolumeGroup

Check the status of the CephFilesystemSubVolumeGroup:

```bash
kubectl -n rook-ceph get cephfilesystemsubvolumegroup
```

Inspect details from the Ceph CLI:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph fs subvolumegroup ls myfs
```

Expected output:

```text
[
    {
        "name": "group1"
    },
    {
        "name": "team-a"
    }
]
```

## Summary

CephFilesystemSubVolumeGroup CRDs in Rook enable declarative management of CephFS subvolume groups, providing logical organization, pinning strategies, and quota controls. By referencing a subvolume group in your StorageClass, you can ensure volumes for different teams or applications are organized and isolated within a shared CephFS filesystem.
