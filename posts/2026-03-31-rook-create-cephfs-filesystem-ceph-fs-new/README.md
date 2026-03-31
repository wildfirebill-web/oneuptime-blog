# How to Create a CephFS Filesystem with ceph fs new

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, CephFS

Description: Learn how to create a CephFS filesystem using the ceph fs new command, configure metadata and data pools, and manage MDS daemons in Rook clusters.

---

## What Is ceph fs new

`ceph fs new` creates a new CephFS filesystem by associating an existing metadata pool with one or more data pools. CephFS requires exactly one metadata pool and at least one data pool. Once created, the filesystem is served by MDS (Metadata Server) daemons.

## Prerequisites

Before creating a CephFS filesystem, create the required pools:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash
```

```bash
# Create metadata pool (replicated, small - metadata is usually small)
ceph osd pool create myfs-metadata 16 replicated
ceph osd pool set myfs-metadata size 3

# Create data pool (this stores the actual file data)
ceph osd pool create myfs-data 64 replicated
ceph osd pool set myfs-data size 3
```

## Creating the Filesystem

Use `ceph fs new` with the filesystem name, metadata pool, and data pool:

```bash
ceph fs new myfs myfs-metadata myfs-data
```

Expected output:

```text
new fs with metadata pool 1 and data pool 2
```

## Verifying the Filesystem

Confirm the filesystem was created:

```bash
ceph fs ls
```

Sample output:

```text
name: myfs, metadata pool: myfs-metadata, data pools: [myfs-data ]
```

Check detailed status:

```bash
ceph fs status myfs
```

## Creating in Rook via CephFilesystem CRD

In Rook, the preferred way to create a CephFS filesystem is via the `CephFilesystem` CRD:

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
  preserveFilesystemOnDelete: true
  metadataServer:
    activeCount: 1
    activeStandby: true
```

Apply it:

```bash
kubectl apply -f cephfilesystem.yaml
```

Rook automatically creates the pools and runs `ceph fs new` on your behalf.

## Creating with Erasure-Coded Data Pool

For large data volumes, use an erasure-coded data pool:

```bash
# Create erasure coding profile
ceph osd erasure-code-profile set myfs-ec k=2 m=1 crush-failure-domain=host

# Create EC data pool
ceph osd pool create myfs-ec-data 64 erasure myfs-ec
ceph osd pool set myfs-ec-data allow_ec_overwrites true

# Add EC pool to existing filesystem
ceph fs add_data_pool myfs myfs-ec-data
```

## MDS Daemon Requirements

After creating the filesystem, you need at least one active MDS daemon. In Rook, the `CephFilesystem` CRD manages MDS pods automatically. To verify MDS status:

```bash
ceph mds stat
```

Sample output:

```text
myfs:1 {0=myfs-a=up:active} 1 up:standby
```

This shows one active MDS (`myfs-a`) and one standby.

## Summary

`ceph fs new` creates a CephFS filesystem by linking a metadata pool and data pool. In Rook environments, use the `CephFilesystem` CRD instead of running `ceph fs new` directly - Rook handles pool creation and MDS daemon management automatically. For large-scale deployments, add erasure-coded data pools using `ceph fs add_data_pool` after initial filesystem creation.
