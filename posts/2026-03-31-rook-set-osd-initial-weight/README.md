# How to Set OSD Initial Weight in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, OSD, CRUSH, Rebalancing

Description: Learn how to set and manage OSD initial CRUSH weight in Ceph to control data distribution and minimize rebalancing when adding new OSDs.

---

## Understanding OSD Weight in Ceph

Every OSD in a Ceph cluster has a CRUSH weight that determines how much data it stores relative to other OSDs. By default, CRUSH assigns weight proportional to the OSD's raw disk capacity in terabytes: a 2 TB drive gets weight 2.0 and a 4 TB drive gets weight 4.0. When a new OSD is added, CRUSH immediately begins moving data to fill it according to its weight, which can cause a rebalancing storm if the full weight is applied at once.

## Checking Current OSD Weights

View the CRUSH map weights for all OSDs:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd tree
```

The `WEIGHT` column shows the CRUSH weight. The `REWEIGHT` column shows the reweight (a multiplier applied on top of CRUSH weight).

## Starting New OSDs with Zero Weight

To add an OSD without immediately triggering rebalancing, Rook supports the `startWithZeroWeight` option in the `CephCluster` spec:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  storage:
    config:
      osdsPerDevice: "1"
    nodes:
    - name: worker-4
      devices:
      - name: sdb
        config:
          initialWeight: "0"
```

The OSD joins the cluster but receives no data until you manually set its weight.

## Gradually Increasing OSD Weight

Incrementally increase the weight to spread rebalancing over time:

```bash
# Set weight to 25% of target
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd reweight osd.9 0.25

# Wait for cluster to stabilize, then increase
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd reweight osd.9 0.5

# Continue incrementally
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd reweight osd.9 0.75

# Final weight
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd reweight osd.9 1.0
```

## Setting CRUSH Weight Directly

CRUSH weight is different from reweight. To change the underlying CRUSH weight:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd crush reweight osd.9 2.0
```

This sets the absolute CRUSH weight to 2.0 (suitable for a 2 TB drive).

## Monitoring Rebalancing Progress

After each weight change, wait for the cluster to stabilize before increasing further:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  watch -n 10 "ceph status | grep -E 'misplaced|backfill|recovery'"
```

When `misplaced` objects reach 0, rebalancing is complete and you can proceed to the next increment.

## Summary

Setting OSD initial weight to zero prevents rebalancing storms when adding new nodes. Gradually increasing weight via `ceph osd reweight` spreads the data movement over time, keeping client I/O latency predictable. For permanent weight changes, use `ceph osd crush reweight` to update the CRUSH map directly. Always monitor misplaced object counts between increments.
