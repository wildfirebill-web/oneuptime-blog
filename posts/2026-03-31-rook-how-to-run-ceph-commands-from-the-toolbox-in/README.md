# How to Run Ceph Commands from the Toolbox in Rook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Toolbox, Cli, Administration, Kubernetes

Description: Learn the most useful Ceph CLI commands to run from the Rook toolbox pod for monitoring cluster health, managing pools, and diagnosing issues.

---

## Overview

The Rook toolbox pod provides access to the full suite of Ceph CLI tools. This guide covers the most commonly used commands for cluster monitoring, OSD management, pool administration, and troubleshooting.

## Cluster Health and Status

Check overall cluster health:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph status
```

Get detailed health warnings and errors:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph health detail
```

Watch cluster status in real time:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph -w
```

## Monitor Commands

List monitors and their status:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph mon stat
```

Dump the monitor map:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph mon dump
```

## OSD Commands

List all OSDs and their state:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd status
```

Get OSD utilization:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd df tree
```

Expected output showing per-OSD disk usage:

```text
ID  CLASS  WEIGHT   REWEIGHT  SIZE     USE      DATA     OMAP     META     AVAIL    %USE  VAR  PGS  STATUS
 0    hdd  0.97656   1.00000  1000 GiB  1.8 GiB  1.7 GiB     0 B  100 MiB  998 GiB  0.18  0.95  32  up
 1    hdd  0.97656   1.00000  1000 GiB  1.9 GiB  1.8 GiB     0 B  100 MiB  998 GiB  0.19  1.00  33  up
```

Mark an OSD out for maintenance:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd out osd.2
```

Mark it back in after maintenance:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd in osd.2
```

## Pool Commands

List all pools:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd pool ls detail
```

Get pool statistics:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph df
```

Get per-pool I/O statistics:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd pool stats
```

## MDS Commands (CephFS)

Check MDS status:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph mds stat
```

Get filesystem status:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph fs status myfs
```

## RBD Commands

List RBD images in a pool:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- rbd ls replicapool
```

Get info for a specific image:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rbd info replicapool/my-image
```

Check RBD mirror status:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rbd mirror pool status replicapool
```

## Object Store (radosgw-admin) Commands

List all RGW users:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  radosgw-admin user list
```

Get user info and quota usage:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  radosgw-admin user info --uid=my-user
```

List all buckets:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  radosgw-admin bucket list
```

## Placement Group Commands

Show PG summary:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph pg stat
```

List stuck PGs:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph pg dump_stuck
```

## Summary

The Ceph toolbox provides access to all Ceph CLI commands through `kubectl exec`. Essential commands include `ceph status` for cluster health, `ceph osd df tree` for capacity planning, `ceph fs status` for CephFS monitoring, and `radosgw-admin` for object store management. These commands are the primary interface for operational tasks that fall outside the Kubernetes CRD model used by Rook.
