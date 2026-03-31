# How to Configure CephX for MDS Authentication

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, CephX, MDS, CephFS, Authentication

Description: Configure CephX authentication for the Ceph Metadata Server (MDS) to control which clients can mount CephFS and access specific directories or filesystem paths.

---

The Ceph Metadata Server (MDS) manages CephFS directory metadata. CephX authentication for MDS controls which clients can mount the filesystem and which paths they can access. Path-based restrictions are a powerful feature for multi-tenant CephFS deployments.

## MDS Capability Syntax

MDS capabilities control filesystem access:

```text
allow [rwp] [path=<path>] [uid=<uid>] [gid=<gid>]
```

| Capability | Meaning |
|------------|---------|
| `allow r` | Read-only mount |
| `allow rw` | Read-write mount |
| `allow rwp` | Read-write with policy capabilities |
| `allow r path=/data` | Read-only access to /data only |

## Create a CephFS Client Key

Create a key for an application that needs full read-write access to CephFS:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph auth get-or-create client.cephfs-app \
  mon 'allow r' \
  mds 'allow rw' \
  osd 'allow rw pool=cephfs-data, allow rw pool=cephfs-metadata' \
  mgr 'allow r'
```

Create a read-only key for a backup service:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph auth get-or-create client.cephfs-backup \
  mon 'allow r' \
  mds 'allow r' \
  osd 'allow r pool=cephfs-data, allow r pool=cephfs-metadata'
```

## Restrict Access to a Specific Path

Limit a client to a specific CephFS directory:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph auth get-or-create client.tenant-a \
  mon 'allow r' \
  mds 'allow rw path=/tenants/tenant-a' \
  osd 'allow rw pool=cephfs-data, allow rw pool=cephfs-metadata'
```

## Use the Key to Mount CephFS in Kubernetes

Store the MDS key as a Kubernetes Secret:

```bash
KEY=$(kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph auth get-key client.cephfs-app)

kubectl -n myapp create secret generic ceph-secret \
  --from-literal=key="$KEY"
```

Reference in a PersistentVolume:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: cephfs-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
  - ReadWriteMany
  csi:
    driver: cephfs.csi.ceph.com
    nodeStageSecretRef:
      name: ceph-secret
      namespace: myapp
    volumeAttributes:
      monitors: "mon1:6789,mon2:6789"
      rootPath: /
```

## Verify MDS Key Permissions

Confirm the key has correct MDS capabilities:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph auth get client.cephfs-app
```

Check MDS status and active filesystem:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph fs status
```

## Summary

MDS CephX capabilities support path-based access restrictions, making it possible to isolate different tenants or applications to specific CephFS subdirectories. Create separate client keys for each application using `ceph auth get-or-create`, restrict MDS access with `path=` in the capability string, and store keys as Kubernetes Secrets for consumption by the CephFS CSI driver.
