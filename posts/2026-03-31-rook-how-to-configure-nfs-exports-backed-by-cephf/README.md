# How to Configure NFS Exports Backed by CephFS in Rook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Nfs, Cephfs, Kubernetes, Exports

Description: Learn how to configure NFS exports backed by CephFS in Rook, enabling NFS clients to access CephFS directories with customizable access controls and options.

---

## Overview

Once a CephNFS server is deployed via Rook, you can configure NFS exports that expose specific CephFS paths to NFS clients. Each export maps a pseudo-path (the NFS mount path clients use) to an actual CephFS path, with configurable access controls and export options.

## Prerequisites

Ensure you have:
- A running CephNFS cluster (e.g., `my-nfs`)
- A deployed CephFilesystem (e.g., `myfs`)

```bash
kubectl -n rook-ceph get cephnfs,cephfilesystem
```

## Creating an Export via the Ceph Dashboard

The Ceph Dashboard provides a GUI for managing NFS exports. Access it by port-forwarding:

```bash
kubectl -n rook-ceph port-forward svc/rook-ceph-mgr-dashboard 7000
```

Navigate to `https://localhost:7000` and go to NFS - Exports - Create.

## Creating Exports via the Ceph CLI

Use the toolbox to create exports programmatically:

```bash
# Export the root of myfs
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph nfs export create cephfs \
    --cluster-id my-nfs \
    --pseudo-path /cephfs \
    --fsname myfs \
    --path / \
    --readonly false

# Export a specific subdirectory
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph nfs export create cephfs \
    --cluster-id my-nfs \
    --pseudo-path /team-a \
    --fsname myfs \
    --path /team-a \
    --readonly false
```

## Listing and Inspecting Exports

List all exports for the NFS cluster:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph nfs export ls my-nfs
```

Expected output:

```text
[
    "/cephfs",
    "/team-a"
]
```

Get detailed information for a specific export:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph nfs export get my-nfs /team-a
```

Output showing export configuration:

```text
{
    "export_id": 2,
    "path": "/team-a",
    "cluster_id": "my-nfs",
    "pseudo": "/team-a",
    "access_type": "RW",
    "squash": "no_root_squash",
    "protocols": [4],
    "transports": ["TCP"],
    "fsal": {
        "name": "CEPH",
        "fs_name": "myfs"
    }
}
```

## Configuring Export Access Controls

Update an export's access type or squash settings:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph nfs export update my-nfs /team-a \
  --access-type RO \
  --squash root_squash
```

To allow only specific client IPs to access an export, use a custom GANESHA export config:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph nfs export apply my-nfs - <<EOF
{
  "export_id": 10,
  "path": "/secure-data",
  "cluster_id": "my-nfs",
  "pseudo": "/secure-data",
  "access_type": "RW",
  "squash": "root_squash",
  "fsal": {
    "name": "CEPH",
    "fs_name": "myfs"
  },
  "clients": [
    {
      "addresses": ["10.0.1.0/24"],
      "access_type": "RW",
      "squash": "no_root_squash"
    },
    {
      "addresses": ["0.0.0.0/0"],
      "access_type": "None"
    }
  ]
}
EOF
```

## Mounting the Export

Get the NFS service endpoint:

```bash
kubectl -n rook-ceph get svc -l app=rook-ceph-nfs
```

Mount on a Linux client:

```bash
sudo mount -t nfs4 <nfs-cluster-ip>:/team-a /mnt/team-a
```

Verify the mount:

```bash
df -h /mnt/team-a
```

## Deleting an Export

Remove an export when it is no longer needed:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph nfs export rm my-nfs /team-a
```

## Summary

Configuring NFS exports backed by CephFS in Rook involves using the Ceph CLI or dashboard to create export definitions that map NFS pseudo-paths to CephFS directories. Exports support per-client access controls, squash settings, and read/write permissions. Once configured, standard NFS clients can mount the exported paths, accessing CephFS data through the NFS-Ganesha servers managed by Rook.
