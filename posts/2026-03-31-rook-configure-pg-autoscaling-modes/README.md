# How to Configure PG Autoscaling Modes (Off, On, Warn) in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Placement Group, Autoscaling, Pool

Description: Learn how to configure Ceph PG autoscaling modes - off, on, and warn - for pools in Rook to automate or control Placement Group count management.

---

## PG Autoscaling Overview

Ceph's PG autoscaler (introduced in Nautilus) monitors pool data sizes and adjusts `pg_num` automatically. Three modes control the behavior per pool:

- **`off`**: Autoscaler is disabled. You manage `pg_num` manually.
- **`warn`**: Autoscaler calculates recommendations and emits `POOL_TOO_MANY_PGS` or `POOL_TOO_FEW_PGS` health warnings, but does not change `pg_num`.
- **`on`**: Autoscaler automatically adjusts `pg_num` when the recommendation deviates significantly (by a factor of 3) from the current value.

## Setting Global Default Mode

Configure the cluster-wide default autoscaling mode:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set global osd_pool_default_pg_autoscale_mode on
```

This applies to all new pools. Existing pools retain their current mode.

## Setting Mode Per Pool

Override the mode for a specific pool:

```bash
# Enable automatic scaling for a specific pool
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool set mypool pg_autoscale_mode on

# Use warn mode to review changes before applying
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool set mypool pg_autoscale_mode warn

# Disable autoscaling for manual control
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool set mypool pg_autoscale_mode off
```

## Configuring in Rook CephBlockPool

Set PG autoscaling mode declaratively in the pool CRD:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: replicapool
  namespace: rook-ceph
spec:
  replicated:
    size: 3
  parameters:
    pg_autoscale_mode: "on"
    target_size_ratio: "0.2"
```

The `target_size_ratio` tells the autoscaler that this pool should use 20% of the cluster's total capacity.

## Viewing Current Autoscale Settings

Check autoscaling mode and status for all pools:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool autoscale-status
```

Output columns include:
- `POOL`: pool name
- `SIZE`: current logical data size
- `TARGET SIZE`: target size hint (if set)
- `RATE`: effective replication multiplier
- `RAW CAPACITY`: estimated raw capacity for this pool
- `RATIO`: fraction of cluster capacity
- `TARGET RATIO`: configured target ratio
- `EFFECTIVE RATIO`: used ratio
- `BIAS`: weighting multiplier
- `PG_NUM`: current PG count
- `NEW PG_NUM`: recommended PG count
- `AUTOSCALE`: current mode

## When to Use Each Mode

Use `warn` mode for:
- First deploying a cluster where you want to review recommendations
- Pools with predictable sizes where you prefer manual control
- Compliance environments requiring change approval

Use `on` mode for:
- Long-running pools with variable data growth
- Development and testing clusters
- When you trust Ceph's capacity estimates

## Summary

PG autoscaling modes give flexible control over how Ceph manages Placement Group counts. `on` mode automates PG management for growing workloads. `warn` mode surfaces recommendations without applying them, ideal for change-controlled environments. Configure globally via `osd_pool_default_pg_autoscale_mode` or per-pool in Rook's `CephBlockPool` CRD using the `pg_autoscale_mode` parameter.
