# How to Handle Spanning Pool Issues in PG Autoscaling

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Placement Group, Autoscaling, Pool

Description: Learn how to handle spanning pool issues in Ceph PG autoscaling, where pools share CRUSH rules and autoscaling recommendations conflict.

---

## What Are Spanning Pools?

In Ceph, multiple pools can share the same CRUSH rule. When this happens, the PG autoscaler must consider all pools using a rule together when calculating PG recommendations, because their PGs compete for the same OSDs. This is called a "spanning pool" scenario.

Problems arise when:
- Pools using the same CRUSH rule have conflicting `target_size_ratio` values that add up to more than 1.0
- The autoscaler cannot independently scale one pool without affecting others
- PG recommendations for one pool invalidate recommendations for sibling pools

## Identifying Spanning Pool Issues

Check the autoscaler status for warnings:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool autoscale-status
```

Look for health warnings:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph health detail | grep -i "pool\|pg\|autoscal"
```

Identify which pools share CRUSH rules:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool ls detail | grep crush_rule
```

## Checking CRUSH Rule Assignments

View the CRUSH rule each pool uses:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd dump | grep -E "pool|crush"
```

Pools with the same `crush_rule` number share a rule and span together.

## Resolving Overlapping Target Ratios

If spanning pools have ratios summing above 1.0, the autoscaler normalizes them. To explicitly manage this, use `target_size_bytes` instead of `target_size_ratio` for pools that should have fixed sizes:

```bash
# Pool A: fixed 2 TiB allocation
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool set pool-a target_size_bytes 2199023255552

# Pool B: use remaining capacity proportionally
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool set pool-b target_size_ratio 0.5
```

## Separate CRUSH Rules for Independent Scaling

The cleanest solution is giving each pool its own CRUSH rule so they do not span:

```bash
# Create a new CRUSH rule for pool-b
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd crush rule create-replicated rule-pool-b default host

# Assign pool-b to its own rule
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool set pool-b crush_rule rule-pool-b
```

In Rook, specify the CRUSH rule in the pool CRD:

```yaml
spec:
  replicated:
    size: 3
    requireSafeReplicaSize: true
  parameters:
    crush_rule: rule-pool-b
    pg_autoscale_mode: "on"
    target_size_ratio: "0.5"
```

## Using the bulk Flag for Large Pools

Mark large bulk-storage pools to help the autoscaler prioritize them:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool set pool-a bulk true
```

The `bulk` flag tells the autoscaler this pool should receive more PGs relative to other pools sharing the same rule.

## Summary

Spanning pool issues in PG autoscaling occur when multiple pools share a CRUSH rule with conflicting capacity hints. Resolve by using `target_size_bytes` for pools with fixed sizes, creating dedicated CRUSH rules to prevent spanning, or using the `bulk` flag to guide prioritization. Separate CRUSH rules are the most reliable long-term solution for independent pool scaling.
