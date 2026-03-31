# How to Understand User Types (Individual vs System) in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, Authentication

Description: Understand the difference between individual and system user types in Ceph authentication and when to use each in your Rook-managed cluster.

---

## CephX User Types Overview

Ceph uses CephX for authentication. Every entity that accesses a Ceph cluster - whether a Ceph daemon, an application, or an administrator - has an identity expressed as a typed user. Ceph distinguishes between two broad categories of users: system users (internal daemons) and individual users (external clients and applications).

## System Users (Internal Daemons)

System users are Ceph daemons that authenticate to the cluster as part of normal cluster operation. These users are created automatically when daemons are initialized and follow strict naming conventions.

System user types:

```text
osd.0        - OSD daemon ID 0
mon.node1    - Monitor on host node1
mds.0        - MDS daemon rank 0
mgr.node1    - Manager on host node1
```

System users have capabilities that match their operational role. For example, OSD daemons need broad OSD and monitor capabilities:

```bash
ceph auth get osd.0
```

Sample output:

```text
[osd.0]
    key = AQA...==
    caps mon = "allow profile osd"
    caps osd = "allow *"
    caps mgr = "allow profile osd"
```

You should never manually modify system user capabilities unless recovering from a serious misconfiguration.

## Individual Users (Client and Application Users)

Individual users are created manually to give external applications, administrators, or services access to the cluster. They belong to the `client` type by convention and use the format `client.<id>`.

```text
client.admin       - Full admin user, created by default
client.myapp       - Application user
client.monitoring  - Read-only monitoring user
```

Create an individual user with restricted permissions:

```bash
ceph auth get-or-create client.myapp \
  mon 'allow r' \
  osd 'allow rw pool=appdata'
```

Inspect it:

```bash
ceph auth get client.myapp
```

Sample output:

```text
[client.myapp]
    key = AQB...==
    caps mon = "allow r"
    caps osd = "allow rw pool=appdata"
```

## The client.admin User

The `client.admin` user is a special individual user created automatically at cluster bootstrap. It has full administrative capabilities:

```bash
ceph auth get client.admin
```

```text
[client.admin]
    key = AQC...==
    caps mds = "allow *"
    caps mgr = "allow *"
    caps mon = "allow *"
    caps osd = "allow *"
```

In Rook deployments, the `client.admin` keyring is stored as a Kubernetes Secret:

```bash
kubectl -n rook-ceph get secret rook-ceph-admin-keyring -o yaml
```

## Listing Users by Type

Use `ceph auth ls` and filter by type:

```bash
# List all OSD system users
ceph auth ls | grep "^osd\."

# List all client users
ceph auth ls | grep "^client\."
```

## Rook-Created Users

Rook automatically creates several client users for its internal components:

```bash
ceph auth ls | grep "client.rook"
```

These include users for the CSI driver, crash collector, and monitoring. Do not delete or modify these unless directed by Rook documentation.

## Summary

Ceph distinguishes between system users (OSD, MON, MDS, MGR daemons) and individual users (client.* entries for applications and admins). System users are managed automatically by Ceph and should not be modified. Individual users are created by operators and should follow least-privilege principles - grant only the pool access and capability levels needed for each specific use case.
