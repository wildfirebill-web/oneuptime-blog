# How to Adjust OSD Weights in CRUSH

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, CRUSH, OSD, Storage

Description: Learn how to adjust CRUSH weights and reweights for Ceph OSDs to balance data distribution across drives of different sizes or utilization levels.

---

## Understanding OSD Weights in Ceph

Ceph uses two types of weights for OSDs:

- **CRUSH weight** (`crush_weight`) - relative capacity weight in the CRUSH map, typically set to the OSD size in TB. Determines how much data CRUSH targets for that OSD.
- **Reweight** (`reweight`) - a 0.0 to 1.0 multiplier applied on top of the CRUSH weight. Used for temporary or fine-grained adjustments without modifying the CRUSH map.

The effective data load on an OSD is proportional to `crush_weight * reweight`.

## Viewing Current Weights

```bash
# Show CRUSH weights and reweights for all OSDs
ceph osd tree

# Show utilization alongside weights
ceph osd df tree

# Show only utilization
ceph osd df
```

Output from `ceph osd df`:

```text
ID  CLASS  WEIGHT  REWEIGHT  SIZE    RAW USE  DATA     OMAP  META   AVAIL   %USE  VAR   PGS  STATUS
 0  hdd    1.000   1.00000   1 TiB   450 GiB  440 GiB   ...         570 GiB 43.9  1.10  128  up
 1  hdd    1.000   1.00000   1 TiB   350 GiB  340 GiB   ...         670 GiB 34.2  0.86  99   up
```

`VAR` above 1.0 means the OSD is carrying more than its fair share of data.

## Adjusting CRUSH Weight

Use CRUSH reweight to change the long-term weight based on actual capacity:

```bash
# Set CRUSH weight to match OSD capacity (in TB)
ceph osd crush reweight osd.0 2.0  # 2TB drive

# Set CRUSH weight for multiple OSDs
ceph osd crush reweight osd.0 2.0
ceph osd crush reweight osd.1 1.0
ceph osd crush reweight osd.2 4.0  # 4TB drive

# Verify the new weights
ceph osd tree
```

## Adjusting Reweight (Fine-Tuning)

For temporary adjustments or fine-tuning without changing the CRUSH map:

```bash
# Reduce load on a specific OSD (make it receive less data)
ceph osd reweight osd.0 0.8

# Increase load (restore to full weight)
ceph osd reweight osd.0 1.0

# Set reweight to zero (drain an OSD without removing it from CRUSH)
ceph osd reweight osd.0 0.0
```

## Reweighting by Utilization

Automatically reweight all OSDs based on current disk utilization:

```bash
# Reweight OSDs that are more than 20% above/below average utilization
ceph osd reweight-by-utilization 120

# Preview without applying (dry run)
ceph osd test-reweight-by-utilization 120

# Reweight considering only PG count deviations
ceph osd reweight-by-pg 120
```

## Setting Weights During OSD Creation

When adding OSDs to CRUSH manually, set the weight at creation time:

```bash
# Add OSD with correct capacity weight
ceph osd crush add osd.5 2.0 root=default rack=rack1 host=node-02

# Verify
ceph osd tree | grep osd.5
```

## Automating Weight Corrections with the Balancer

For ongoing automatic weight management:

```bash
# Enable the balancer in upmap mode
ceph balancer mode upmap
ceph balancer on

# Check the score (0 = perfect balance, higher = more imbalance)
ceph balancer eval

# The balancer will continuously adjust to minimize the score
ceph balancer status
```

## Summary

Adjusting OSD weights in Ceph involves two levers: CRUSH weight (long-term capacity-based allocation set with `ceph osd crush reweight`) and reweight (short-term multiplier set with `ceph osd reweight`). For capacity-matched clusters, set CRUSH weights equal to drive sizes in TB. Use reweight for temporary drain operations or fine-tuning. Enable the balancer module for automated ongoing distribution optimization.
