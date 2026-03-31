# How to Manage Pools from the Ceph Dashboard

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Dashboard, Pool, Storage

Description: Create, configure, and monitor Ceph storage pools using the Ceph Dashboard web interface, including replication settings and PG autoscaling.

---

## Overview

Ceph pools are logical partitions of storage that define replication, placement, and performance policies. The Dashboard's Pool section provides a GUI for creating and managing pools without needing CLI commands.

## Accessing the Pool Manager

Navigate to the Pools section:

```bash
# Access dashboard
kubectl -n rook-ceph port-forward svc/rook-ceph-mgr-dashboard 8443:8443
# Navigate to: https://localhost:8443/#/pool
```

The pool list shows:
- Pool name and ID
- Pool type (replicated or erasure-coded)
- Replica count or EC profile
- PG count and target PGs
- Data usage and available space
- Read/write statistics

## Creating a Pool via Dashboard

Click "Create" in the Pools section. Required fields:

- **Name**: unique pool name (lowercase, hyphens allowed)
- **Pool type**: Replicated or Erasure Coded
- **Replicated size**: 2 (testing) or 3 (production)
- **PG Autoscaling**: recommended to enable (set to "on")

For erasure-coded pools:
- Select an existing EC profile or create one
- EC profiles define data chunks and coding chunks

CLI equivalent for reference:

```bash
# Replicated pool
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  ceph osd pool create mypool 32 32 replicated --autoscale-mode on

# Erasure-coded pool
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  ceph osd erasure-code-profile set myec k=4 m=2
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  ceph osd pool create myecpool 32 32 erasure myec --autoscale-mode on
```

## Editing Pool Settings

Click on a pool name, then the Edit button. Common settings to change:

- **Compression**: none / snappy / zlib / zstd (per pool)
- **Quota**: set max bytes or max objects
- **PG Autoscaling mode**: on / warn / off
- **Application tags**: rbd, cephfs, rgw (required for CSI)

Enable compression via CLI:

```bash
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  ceph osd pool set mypool compression_mode aggressive

kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  ceph osd pool set mypool compression_algorithm snappy
```

## Pool Statistics View

The Dashboard pool detail view shows per-pool metrics:
- Bytes used, objects stored
- Read/write IOPS and throughput
- PG distribution across OSDs

Access the same data via CLI:

```bash
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph osd pool stats
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph df detail
```

## Setting Pool Quotas

Prevent a single pool from consuming all cluster capacity:

```bash
# Set max 1TB per pool
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  ceph osd pool set-quota mypool max_bytes 1099511627776

# Verify quota
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  ceph osd pool get-quota mypool
```

## Deleting a Pool

The Dashboard requires a confirmation string before deletion. CLI equivalent:

```bash
# Enable pool deletion (disabled by default)
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  ceph config set mon mon_allow_pool_delete true

# Delete pool
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  ceph osd pool delete mypool mypool --yes-i-really-really-mean-it
```

## Summary

The Ceph Dashboard provides a GUI for creating, editing, and monitoring pools without requiring CLI expertise. Key management tasks include setting replication size, enabling PG autoscaling, configuring compression, and setting quotas. All dashboard actions have CLI equivalents for automation and scripting scenarios.
