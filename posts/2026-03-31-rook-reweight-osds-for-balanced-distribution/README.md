# How to Reweight OSDs for Balanced Data Distribution in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, OSD, Reweight, Distribution

Description: Learn how to reweight Ceph OSDs to correct uneven data distribution and ensure balanced utilization across all storage devices.

---

## Why Reweight OSDs

CRUSH assigns data to OSDs based on their configured weight, which normally corresponds to their capacity in terabytes. If OSDs have different capacities, or if you want to reduce load on a specific OSD before decommissioning, you can adjust weights.

There are two types of reweighting in Ceph:
- **CRUSH weight** - changes the target distribution for all data on the OSD
- **OSD reweight** - a secondary multiplier that adjusts how much data lands on an OSD without editing the CRUSH map

## Checking Current Weights and Utilization

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd df
```

The output shows `WEIGHT` (CRUSH weight), `REWEIGHT`, and `VAR` (variance from average). High variance values indicate imbalance.

## Using OSD Reweight

OSD reweight is a fast way to redistribute data without editing the CRUSH map. Values range from 0.0 (move all data off) to 1.0 (normal weight):

```bash
# Reduce data on osd.5 to 80% of its normal share
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd reweight 5 0.8
```

To remove all data from an OSD (before removal):

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd reweight 5 0.0
```

## Reweight by Utilization

Ceph can automatically reweight OSDs based on their current utilization to bring high-usage OSDs down:

```bash
# Preview what would change
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd test-reweight-by-utilization

# Apply the reweighting
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd reweight-by-utilization
```

By default, only OSDs using more than 20% above average are reweighted. Adjust the threshold:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd reweight-by-utilization 115
```

This reweights OSDs that are 15% above average (115 means 115% of average).

## Changing CRUSH Weight

To change an OSD's CRUSH weight (affects long-term placement):

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd crush reweight osd.5 3.0
```

The weight is typically the OSD's capacity in terabytes. Changing CRUSH weight triggers a full rebalance of data across the affected PGs.

## Monitoring Rebalancing Progress

After reweighting, watch data migration:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph -w | grep -E "backfill|recovery|misplaced"
```

Check recovery speed:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph -s | grep recovering
```

## Applying noout During Reweight

Set `noout` before reweighting to prevent the cluster from treating slow recovery as OSD failures:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd set noout
# Perform reweighting
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd reweight-by-utilization
# Wait for recovery to complete, then clear
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd unset noout
```

## Summary

OSD reweighting controls how much data lands on each OSD without requiring CRUSH map edits. Use `ceph osd reweight` for quick adjustments and `ceph osd reweight-by-utilization` for automatic balancing based on current usage. Always monitor recovery progress after reweighting and use `noout` to prevent false failure detection during migration.
