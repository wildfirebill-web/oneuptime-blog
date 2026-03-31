# How to Set Up CephFS for NFS Re-Export

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, CephFS, NFS, Storage

Description: Set up CephFS NFS re-export in Rook to provide NFSv4 access to CephFS volumes for non-Kubernetes clients and legacy applications.

---

## Why NFS Re-Export CephFS

Not all clients can use the CephFS native protocol or Kubernetes CSI. Legacy applications, Windows clients via NFS, and HPC clusters that cannot run the Ceph kernel module can still access CephFS data through an NFS re-export layer. Rook supports this via a CephNFS resource backed by NFS-Ganesha.

## Prerequisites

Ensure your Rook cluster is running a CephFS filesystem. Then create a CephNFS gateway:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephNFS
metadata:
  name: my-nfs
  namespace: rook-ceph
spec:
  rados:
    pool: nfs-ganesha
    namespace: nfs-ns
  server:
    active: 2
    resources:
      requests:
        cpu: "1"
        memory: "1Gi"
      limits:
        cpu: "2"
        memory: "2Gi"
    placement:
      podAntiAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
          - topologyKey: kubernetes.io/hostname
            labelSelector:
              matchLabels:
                app: rook-ceph-nfs
```

Create the NFS RADOS pool first:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool create nfs-ganesha 32
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool application enable nfs-ganesha nfs
```

## Configuring NFS Exports

Create an NFS export that maps a CephFS path:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph nfs export create cephfs \
    --cluster-id my-nfs \
    --pseudo-path /exports/data \
    --fsname cephfs \
    --path /shared/data
```

List exports:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph nfs export ls my-nfs
```

## Kubernetes Service for NFS

Expose the NFS gateway as a Kubernetes LoadBalancer or NodePort service:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: rook-ceph-nfs
  namespace: rook-ceph
spec:
  selector:
    app: rook-ceph-nfs
    ceph_nfs: my-nfs
  ports:
    - name: nfs
      port: 2049
      protocol: TCP
    - name: rpcbind
      port: 111
      protocol: TCP
  type: LoadBalancer
```

## Mounting from External Clients

From a Linux client outside the Kubernetes cluster:

```bash
mount -t nfs4 <loadbalancer-ip>:/exports/data /mnt/cephfs-nfs \
  -o vers=4.1,proto=tcp,noatime
```

For automount via `/etc/fstab`:

```
<lb-ip>:/exports/data  /mnt/cephfs-nfs  nfs4  vers=4.1,proto=tcp,noatime,_netdev  0  0
```

## Export Permissions and Squash

Control access in the export configuration:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph nfs export create cephfs \
    --cluster-id my-nfs \
    --pseudo-path /exports/readonly \
    --fsname cephfs \
    --path /shared/readonly \
    --readonly
```

For root squash, edit the Ganesha configuration:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph nfs export get my-nfs /exports/data
```

## Summary

CephFS NFS re-export via Rook's CephNFS resource provides a standards-compliant NFSv4.1 interface to CephFS data without requiring Ceph client libraries on external hosts. Running multiple active NFS-Ganesha servers ensures high availability and distributes NFS client connections across pods.
