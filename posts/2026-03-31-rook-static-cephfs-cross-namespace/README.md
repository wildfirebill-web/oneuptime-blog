# How to Set Up Static CephFS Volumes for Cross-Namespace Sharing in Rook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, CephFS, Kubernetes, Storage

Description: Learn how to create static CephFS PersistentVolumes in Rook to share the same filesystem path across multiple Kubernetes namespaces.

---

## Why Static Volumes for Cross-Namespace Sharing

Dynamic provisioning in Kubernetes creates a separate PVC per namespace, each backed by a unique CephFS subvolume. This means two namespaces cannot share the same directory or data.

Static provisioning lets you pre-create a CephFS path and bind multiple PVs across namespaces to that same path. This is useful for:

- Shared configuration or asset directories accessed by multiple teams.
- Read-only reference datasets consumed by many workloads.
- Multi-tenant scenarios where a central directory is exposed to several namespaces.

## Prerequisites

You need an existing CephFS filesystem in your Rook cluster. Confirm it is ready:

```bash
kubectl -n rook-ceph get cephfilesystem
```

You also need the CSI `ceph-admin` secret and cluster ID. Retrieve the cluster ID:

```bash
kubectl -n rook-ceph get configmap rook-ceph-mon-endpoints -o jsonpath='{.data.csi-cluster-config-json}' | python3 -m json.tool
```

## Creating the Shared Directory

Use the Rook toolbox to create a directory inside CephFS:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash
# Inside toolbox:
mkdir -p /mnt/cephfs/shared-data
ceph fs subvolumegroup create myfs shared-group
ceph fs subvolume create myfs shared-vol shared-group
ceph fs subvolume getpath myfs shared-vol shared-group
```

Note the returned path (e.g., `/volumes/shared-group/shared-vol/...`).

## Creating a Static PersistentVolume

Create a PV that references the CephFS path directly. This PV can be bound in any namespace:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: cephfs-shared-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  csi:
    driver: rook-ceph.cephfs.csi.ceph.com
    nodeStageSecretRef:
      name: rook-csi-cephfs-node
      namespace: rook-ceph
    controllerExpandSecretRef:
      name: rook-csi-cephfs-provisioner
      namespace: rook-ceph
    volumeHandle: cephfs-shared-volume-handle
    volumeAttributes:
      clusterID: "<your-cluster-id>"
      fsName: myfs
      staticVolume: "true"
      rootPath: /volumes/shared-group/shared-vol/<uuid>
```

## Creating PVCs in Multiple Namespaces

In each namespace, create a PVC that binds to the same PV by referencing it explicitly:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-cephfs-pvc
  namespace: team-alpha
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
  volumeName: cephfs-shared-pv
  storageClassName: ""
```

Repeat for `team-beta` namespace. Both PVCs bind to the same PV.

## Important Considerations

The `Retain` reclaim policy is critical. If a PVC is deleted, the PV and its data persist. Set `staticVolume: "true"` in `volumeAttributes` so the CSI driver does not attempt to delete the underlying CephFS path on PV release.

For write access from multiple namespaces simultaneously, ensure your workloads handle concurrent writes safely. CephFS supports `ReadWriteMany`, but application-level locking may still be needed.

## Summary

Static CephFS PersistentVolumes allow sharing a single Ceph filesystem path across multiple Kubernetes namespaces. By pre-creating the subvolume, defining a PV with `staticVolume: "true"`, and binding PVCs from different namespaces to the same PV name, teams can access shared data without duplicating storage. Always use `Retain` reclaim policy to protect data when PVCs are removed.
