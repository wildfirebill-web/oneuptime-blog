# How to Set Up NFSv4 Protocol with Rook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, NFS, Protocol, Kubernetes

Description: Learn how to configure NFSv4 protocol settings for a CephNFS cluster in Rook, including ID mapping and security options.

---

## Why NFSv4 Configuration Matters

Rook's CephNFS uses NFS-Ganesha as the userspace NFS server. By default, it supports NFSv3 and NFSv4. For production Kubernetes environments, NFSv4 is preferred because it provides stateful connections, better locking semantics, and support for Kerberos-based security. Configuring NFSv4 correctly in Rook involves setting the NFS version constraints in the `CephNFS` spec and optionally tuning idmapping and lease times.

## Basic CephNFS with NFSv4 Only

To restrict the server to NFSv4 and disable NFSv3, configure the `CephNFS` CR with an explicit Ganesha config block:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephNFS
metadata:
  name: my-nfs
  namespace: rook-ceph
spec:
  rados:
    pool: my-fs-data0
    namespace: nfs-ns
    object: conf-nfs.my-nfs
  server:
    active: 1
  server:
    active: 1
    logLevel: NIV_INFO
  ganesha:
    config: |
      NFS_CORE_PARAM {
        Protocols = 4;
        NFS_Port = 2049;
      }
      NFSv4 {
        Lease_Lifetime = 90;
        Grace_Period = 90;
        Minor_Versions = 1, 2;
      }
```

Setting `Protocols = 4` ensures only NFSv4 and its minor versions (4.1, 4.2) are served.

## Configuring ID Mapping for NFSv4

NFSv4 uses string-based user/group identities rather than numeric UIDs/GIDs. The `IDMAP` config in Ganesha controls how these map to local IDs:

```yaml
ganesha:
  config: |
    NFS_CORE_PARAM {
      Protocols = 4;
    }
    IDMAPD {
      Domain = cluster.local;
    }
```

Clients must also configure their `idmapd.conf` with a matching domain. On the client node:

```bash
# /etc/idmapd.conf
[General]
Domain = cluster.local
```

Restart the idmapd service on the client after changing this:

```bash
systemctl restart nfs-idmapd
```

## Mounting with NFSv4

When mounting from a Kubernetes node or external client, specify NFSv4 explicitly:

```bash
mount -t nfs4 \
  -o proto=tcp,port=2049,vers=4.2 \
  <nfs-service-ip>:/cephfs-export \
  /mnt/nfs4
```

Or use a Kubernetes PersistentVolume with the NFS CSI driver that targets the NFS service:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteMany
  nfs:
    server: rook-ceph-nfs-my-nfs-0.rook-ceph.svc
    path: /cephfs-export
  mountOptions:
    - vers=4.2
    - proto=tcp
```

## Verifying NFSv4 Connections

Check active NFSv4 state from the Rook toolbox:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rpcinfo -p <nfs-pod-ip>
```

Also inspect Ganesha logs for version negotiation:

```bash
kubectl -n rook-ceph logs \
  $(kubectl -n rook-ceph get pod -l app=rook-ceph-nfs -o name | head -1) \
  | grep "NFS version"
```

## Summary

Configuring NFSv4 with Rook means setting `Protocols = 4` and tuning `NFSv4` lease and grace periods in the Ganesha config block within the `CephNFS` spec. Match the idmap domain between server and clients to ensure correct user/group resolution, and use `vers=4.2` in mount options to take advantage of the latest NFSv4 minor version features.
