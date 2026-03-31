# How to View PG Scaling Recommendations with autoscale-status

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Placement Group, Autoscaling, Pool

Description: Learn how to use ceph osd pool autoscale-status to view PG scaling recommendations and understand why Ceph suggests changing pg_num for your pools.

---

## Why Review PG Scaling Recommendations?

The Ceph PG autoscaler continuously monitors pool sizes and suggests PG count adjustments. Reviewing these recommendations helps you understand whether pools are over or under-provisioned with PGs, catch configuration issues early, and validate autoscaler behavior before enabling automatic changes.

## Running autoscale-status

From the Rook toolbox:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool autoscale-status
```

For formatted output:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool autoscale-status --format json | python3 -m json.tool
```

## Reading the Output

A typical output table looks like:

```text
POOL                     SIZE  TARGET SIZE  RATE  RAW CAPACITY  RATIO  TARGET RATIO  EFFECTIVE RATIO  BIAS  PG_NUM  NEW PG_NUM  AUTOSCALE  BULK
device_health_metrics      0               3.0      11.3T         0.0                          0.0   1.0       1              warn
replicapool               50G               3.0      11.3T         0.013                        0.0   1.0     128      32       warn
.mgr                     1.5M               1.0      11.3T         0.0                          0.0   4.0       1              warn
```

Key columns explained:

- `SIZE`: Actual logical data stored in the pool
- `TARGET SIZE`: Manually configured target size hint (from `target_size_bytes`)
- `RATE`: Replication multiplier (3x for 3-way replication, variable for erasure coding)
- `RATIO`: Current fraction of raw cluster capacity used by this pool
- `PG_NUM`: Current number of PGs
- `NEW PG_NUM`: Recommended PG count (blank means current count is optimal)
- `AUTOSCALE`: Current mode (off/warn/on)

## Interpreting Recommendations

When `NEW PG_NUM` differs significantly from `PG_NUM`, the autoscaler is recommending a change. Ceph only acts (in `on` mode) when the ratio is off by a factor of 3x or more.

A recommendation to decrease PGs means:
- The pool has fewer objects than expected
- Current PG count wastes memory

A recommendation to increase PGs means:
- The pool has grown larger than the initial PG estimate accounted for
- PGs are unbalanced, with some OSDs handling too much data

## Checking Why a Recommendation Was Made

Get detailed reasons for a specific pool:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool autoscale-status --format json | \
  python3 -c "import sys,json; \
  [print(p['pool_name'], p['pg_num'], '->', p.get('pg_num_final','same')) \
  for p in json.load(sys.stdin)]"
```

## Acting on Recommendations Manually

If autoscaling is in `warn` mode, apply recommendations manually:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool set replicapool pg_num 32

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool set replicapool pgp_num 32
```

Always reduce `pg_num` and `pgp_num` together and set `pgp_num` last.

## Summary

`ceph osd pool autoscale-status` is the primary tool for understanding PG scaling recommendations. Review it regularly to validate pool sizing, identify misconfigurations, and understand what the autoscaler would change in `on` mode. Use the `NEW PG_NUM` column as guidance for manual adjustments when running in `warn` mode.
