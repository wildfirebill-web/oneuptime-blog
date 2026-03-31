# How to Set target_size_ratio for PG Autoscaling in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Placement Group, Autoscaling, Pool

Description: Learn how to use target_size_ratio to give the Ceph PG autoscaler a proportional capacity hint, helping allocate PGs correctly across multiple pools.

---

## What Is target_size_ratio?

`target_size_ratio` is an alternative to `target_size_bytes` for guiding the Ceph PG autoscaler. Instead of an absolute byte count, you specify what fraction of the cluster's total raw capacity this pool should eventually consume. This is useful when:

- Cluster size will grow over time (ratio stays valid even as capacity increases)
- You want proportional allocation across multiple pools
- You don't know exact byte sizes but understand relative pool sizes

For example, if you have a 3-pool cluster and want equal data distribution, set each pool's ratio to `0.33`.

## Setting target_size_ratio

From the Rook toolbox:

```bash
# Pool should use 30% of cluster capacity
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool set mypool target_size_ratio 0.3

# Small helper pool using 5%
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool set helper-pool target_size_ratio 0.05
```

## Configuring in Rook CephBlockPool

```yaml
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: primary-pool
  namespace: rook-ceph
spec:
  replicated:
    size: 3
  parameters:
    pg_autoscale_mode: "on"
    target_size_ratio: "0.5"
```

## Multi-Pool Ratio Planning

When setting ratios across multiple pools, they should not exceed 1.0 total (though Ceph will still work if they do - it normalizes the ratios internally):

```bash
# Set ratios for three pools
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool set pool-a target_size_ratio 0.4

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool set pool-b target_size_ratio 0.4

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool set pool-c target_size_ratio 0.2
```

## Verifying Ratio-Based Autoscaling

Check how the autoscaler interprets ratios:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool autoscale-status
```

The `TARGET RATIO` and `EFFECTIVE RATIO` columns show configured and calculated values. The `NEW PG_NUM` column shows the recommended PG count based on the ratio.

## Ratio vs Bytes - Which to Use?

Use `target_size_ratio` when:
- Building a new cluster and planning capacity allocation
- Multiple pools should share cluster capacity proportionally
- The cluster will scale horizontally and ratios stay proportionally valid

Use `target_size_bytes` when:
- A pool has a known maximum size (e.g., backups, fixed datasets)
- You want autoscaling that does not change as cluster capacity grows

## Clearing the Ratio

Remove the ratio hint:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool set mypool target_size_ratio 0
```

## Summary

`target_size_ratio` provides the PG autoscaler with proportional capacity hints, making it ideal for multi-pool clusters with planned capacity allocation. Set ratios that add up to approximately 1.0 across all pools. Use `ceph osd pool autoscale-status` to verify the autoscaler is computing appropriate PG counts from your configured ratios.
