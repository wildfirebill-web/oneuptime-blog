# How to Use the Placement Group Calculator

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Placement Group, Storage, Performance

Description: Use the Ceph placement group calculator to determine the optimal PG count for your pools and avoid common over- or under-provisioning mistakes.

---

## What Is the Placement Group Calculator

Placement Groups (PGs) are the internal sharding unit in Ceph. Each pool has a fixed number of PGs, and Ceph distributes data across OSDs by mapping objects to PGs and then PGs to OSDs. Too few PGs causes uneven data distribution and limits parallelism; too many PGs wastes memory and CPU on each OSD.

The Ceph project provides a web-based PG calculator at `https://old.ceph.com/pgcalc/` and a built-in command to suggest values.

## Inputs to the Calculation

Before using the calculator, gather:

- **Number of OSDs** in the cluster.
- **Replication factor** (or erasure coding parameters) for the pool.
- **Number of pools** you intend to create.
- **Target % of data per OSD** (the calculator uses 100% as the base).

The key formula is:

```
PGs = (OSDs * target_pgs_per_osd) / replication_factor
```

Round the result up to the nearest power of two.

## Using the Built-in Ceph Calculator

From the Rook toolbox, run:

```bash
kubectl -n rook-ceph exec -it <toolbox-pod> -- \
  ceph osd pool get-quota <pool-name>
```

To get an active suggestion:

```bash
kubectl -n rook-ceph exec -it <toolbox-pod> -- \
  ceph osd pool autoscale-status
```

If the autoscaler is enabled, it shows the current PG count vs. the optimal count:

```bash
POOL                     SIZE  TARGET SIZE  RATE  RAW CAPACITY  RATIO  TARGET RATIO  EFFECTIVE RATIO  BIAS  PG_NUM  NEW PG_NUM  AUTOSCALE
replicapool              10G   -            3     300G          0.033  -             -                1.0   32      -           on
```

## Recommended PG Count per OSD

The general guideline is 100-200 PGs per OSD across all pools. For a cluster with 10 OSDs and 3 pools at replication factor 3:

```
total_pgs = 10 * 100 = 1000
pgs_per_pool = 1000 / 3 = 333
```

Round to the nearest power of two: **256** PGs per pool is a good starting point.

## Setting PG Count via Rook

In your `CephBlockPool` manifest:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: replicapool
  namespace: rook-ceph
spec:
  failureDomain: host
  replicated:
    size: 3
  parameters:
    pg_num: "128"
    pgp_num: "128"
```

For large clusters, let the autoscaler manage PG count instead:

```yaml
spec:
  parameters:
    pg_autoscale_mode: "on"
```

## Enabling the PG Autoscaler

The PG autoscaler is a Ceph manager module that adjusts PG counts automatically as data grows. Enable it from the toolbox:

```bash
kubectl -n rook-ceph exec -it <toolbox-pod> -- \
  ceph mgr module enable pg_autoscaler

kubectl -n rook-ceph exec -it <toolbox-pod> -- \
  ceph config set global osd_pool_default_pg_autoscale_mode on
```

## Interpreting Autoscaler Output

Check the autoscaler's recommendations:

```bash
kubectl -n rook-ceph exec -it <toolbox-pod> -- \
  ceph osd pool autoscale-status
```

If `NEW PG_NUM` differs from `PG_NUM`, the autoscaler wants to resize the pool. In `on` mode it does so automatically; in `warn` mode it only alerts.

## Summary

The Ceph PG calculator helps you avoid under- or over-provisioning PGs in your Rook cluster. Start with the rule of thumb of 100 PGs per OSD across all pools, round to a power of two, and set `pg_autoscale_mode: on` so Ceph adjusts PG counts as your data grows. Use `ceph osd pool autoscale-status` from the toolbox to monitor recommendations at any time.
