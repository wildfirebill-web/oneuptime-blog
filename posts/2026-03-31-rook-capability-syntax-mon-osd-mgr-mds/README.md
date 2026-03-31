# How to Understand Capability Syntax (mon, osd, mgr, mds) in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, Authentication

Description: Learn the syntax for Ceph capability strings across the mon, osd, mgr, and mds subsystems to correctly configure user access in Rook clusters.

---

## Overview of Ceph Capability Subsystems

Ceph's CephX authorization system uses capability strings to define what each authenticated entity can do. Each Ceph subsystem (mon, osd, mgr, mds) has its own capability syntax, and a user entry can include caps for any combination of subsystems.

The general form when creating or updating a user:

```bash
ceph auth get-or-create client.example \
  mon '<mon-caps>' \
  osd '<osd-caps>' \
  mgr '<mgr-caps>' \
  mds '<mds-caps>'
```

## Monitor (mon) Capabilities

Monitor capabilities control access to cluster-level operations like reading cluster state, creating pools, and managing auth.

```text
allow r              - Read-only access to monitors (status, pg info, etc.)
allow rw             - Read and write access (create/delete pools, etc.)
allow *              - Full admin access
allow profile osd    - Pre-defined profile for OSD daemons
allow profile mds    - Pre-defined profile for MDS daemons
allow profile mgr    - Pre-defined profile for MGR daemons
```

Example - read-only monitoring client:

```bash
ceph auth get-or-create client.monitor \
  mon 'allow r'
```

## OSD Capabilities

OSD capabilities control access to object storage - pools, namespaces, and objects.

```text
allow r                          - Read from all pools
allow rw                         - Read and write to all pools
allow *                          - Full access to all OSDs
allow rw pool=mypool             - Read-write restricted to one pool
allow r pool=pool1, allow rw pool=pool2   - Mixed access per pool
allow rw namespace=mynamespace   - Restrict to a specific namespace
allow rw tag cephfs data=myfs    - Tag-based access for CephFS pools
profile rbd                      - Pre-defined RBD client profile
profile rbd pool=myrbd           - RBD profile restricted to a pool
```

Example - application user with pool restriction:

```bash
ceph auth get-or-create client.webapp \
  mon 'allow r' \
  osd 'allow rw pool=webapp-data'
```

## Manager (mgr) Capabilities

Manager capabilities control access to the Ceph manager, which handles orchestration, modules, and metrics.

```text
allow r              - Read access to manager
allow rw             - Read-write access to manager
allow *              - Full manager access
allow profile osd    - OSD-related manager access
allow profile rbd    - RBD-related manager access (for mirroring)
```

Example - prometheus monitoring user:

```bash
ceph auth get-or-create client.prometheus \
  mon 'allow r' \
  mgr 'allow r'
```

## MDS (Metadata Server) Capabilities

MDS capabilities control CephFS operations.

```text
allow r              - Read metadata
allow rw             - Read and write metadata
allow *              - Full MDS access
allow rw path=/data  - Restrict access to a specific directory path
```

Example - CephFS client restricted to a path:

```bash
ceph auth get-or-create client.cephfs-app \
  mon 'allow r' \
  mds 'allow rw path=/data/apps' \
  osd 'allow rw tag cephfs data=myfs'
```

## Full Admin User

The admin user has full access to all subsystems:

```bash
ceph auth get-or-create client.admin \
  mon 'allow *' \
  osd 'allow *' \
  mgr 'allow *' \
  mds 'allow *'
```

## Combining Multiple Pool Rules

OSD caps support comma-separated rules for complex access patterns:

```bash
ceph auth get-or-create client.multi \
  mon 'allow r' \
  osd 'allow r pool=readonly, allow rw pool=readwrite, allow rw pool=another'
```

## Verifying Capabilities

After creating users, always inspect with `ceph auth get`:

```bash
ceph auth get client.webapp
```

```text
[client.webapp]
    key = AQD...==
    caps mon = "allow r"
    caps osd = "allow rw pool=webapp-data"
```

## Summary

Ceph capabilities use per-subsystem strings with `allow` clauses. The `mon` subsystem controls cluster-level operations, `osd` controls pool and object access, `mgr` controls manager access, and `mds` controls CephFS operations. Use pool restrictions in OSD caps and path restrictions in MDS caps to enforce least-privilege access for each application or service in your Rook cluster.
