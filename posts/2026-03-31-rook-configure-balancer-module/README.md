# How to Configure the Balancer Module in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Balancer, Performance, Distribution

Description: Learn how to enable and configure the Ceph balancer module to automatically optimize data distribution across OSDs for even utilization.

---

## What is the Ceph Balancer Module

The balancer is a Ceph Manager module that continuously monitors OSD utilization and computes pg-upmap or CRUSH weight adjustments to minimize data distribution variance. An even distribution ensures no single OSD becomes a hotspot.

The balancer works in the background and can run automatically or on-demand.

## Enabling the Balancer Module

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph mgr module enable balancer
```

Verify it is enabled:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph mgr module ls | grep balancer
```

## Balancer Modes

The balancer supports two modes:

- **crush-compat** - adjusts OSD weights in the CRUSH map to balance data
- **upmap** - uses pg-upmap entries to remap individual PGs

`upmap` is more precise and is recommended for Luminous and later clusters:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph balancer mode upmap
```

## Evaluating Current Balance

Before enabling automatic balancing, evaluate the current state:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph balancer eval
```

Output shows the current score (lower is better, 0 is perfect balance):

```text
current score 0.0231
```

Check which pool has the worst balance:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph balancer eval mypool
```

## Running the Balancer On-Demand

Generate and apply an optimization plan manually:

```bash
# Create a named plan
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph balancer optimize myplan

# Review the plan
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph balancer show myplan

# Execute the plan
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph balancer execute myplan

# Discard the plan without executing
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph balancer rm myplan
```

## Enabling Automatic Balancing

Turn on automatic background balancing:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph balancer on
```

The balancer will continuously compute and apply small upmap adjustments to maintain balance. Disable it:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph balancer off
```

## Configuring Balancer Thresholds

Set the minimum improvement threshold before the balancer acts:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set mgr mgr/balancer/min_score 0.02
```

Limit which pools are balanced:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set mgr mgr/balancer/pool_ids "1 2 3"
```

## Verifying Balance After Optimization

Check OSD utilization variance after the balancer runs:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd df
```

The `VAR` column should be close to 1.0 for all OSDs.

## Summary

The Ceph balancer module reduces data distribution variance by automatically computing and applying pg-upmap adjustments. Enable it with `ceph mgr module enable balancer`, set the mode to `upmap`, and turn on automatic balancing with `ceph balancer on`. Monitor effectiveness via `ceph balancer eval` and `ceph osd df`.
