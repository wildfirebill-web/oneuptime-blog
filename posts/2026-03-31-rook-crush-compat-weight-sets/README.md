# How to Configure Compat Weight Sets in CRUSH

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, CRUSH, Weight Set, Storage

Description: Learn how to configure Ceph CRUSH compat weight sets to enable the balancer module to optimize data distribution without changing the CRUSH map structure.

---

## What are CRUSH Weight Sets

CRUSH weight sets are alternative sets of weights that can be assigned to OSDs in the CRUSH map. Instead of using the standard CRUSH weights (based on OSD capacity), weight sets allow the balancer module to use fine-tuned weights that achieve better actual data distribution.

There are two types of weight sets:
- **compat weight set** - a single set of alternative weights shared across all pools, compatible with older clients
- **per-pool weight sets** - separate optimized weights for each pool, requiring newer client support

The compat weight set is the default mode used by Ceph's balancer module when enabled.

## Checking Current Weight Set Status

```bash
# Check if the balancer module is active
ceph mgr module ls | grep balancer
ceph balancer status

# Show current CRUSH weight sets
ceph osd crush dump | python3 -m json.tool | grep -A20 "weight_set"

# Check the CRUSH dump for compat weights
ceph osd getcrushmap -o crush.bin
crushtool -d crush.bin -o crush.txt
grep -A5 "weight_set" crush.txt
```

## Enabling the Balancer with Compat Mode

```bash
# Enable the balancer module
ceph mgr module enable balancer

# Set balancer to use compat weight set mode
ceph balancer mode upmap

# Or use crush-compat mode for older clients
ceph balancer mode crush-compat

# Turn the balancer on
ceph balancer on

# Check status
ceph balancer status
```

## Manually Creating a Compat Weight Set

You can manually create a compat weight set via the CRUSH map:

```bash
ceph osd getcrushmap -o crush.bin
crushtool -d crush.bin -o crush.txt
```

Add a compat weight set section to the decompiled map:

```text
# compat weight set
weight_set {
    # weights for each OSD (by OSD ID order)
    1.000 1.000 1.000 1.000 1.000 1.000
}
```

```bash
crushtool -c crush.txt -o crush-new.bin
ceph osd setcrushmap -i crush-new.bin
```

## Viewing Balancer Recommendations

```bash
# Evaluate current data distribution quality
ceph balancer eval

# Show current deviation from optimal distribution
ceph balancer eval 2>&1

# See what changes the balancer would make
ceph balancer optimize myplan

# Show the optimization plan details
ceph balancer show myplan
```

Example output:

```text
evaluating plan myplan
using crush-compat mode
10 pg movements
score 0.038 -> 0.003
```

## Applying Balancer Changes

```bash
# Execute a balancer plan
ceph balancer execute myplan

# Or let the balancer run automatically
ceph balancer on
ceph balancer status

# Monitor rebalancing progress
watch -n 5 ceph -s
```

## Resetting Weight Sets

```bash
# Remove compat weight sets and return to standard CRUSH weights
ceph balancer off
ceph osd crush weight-set rm *

# Verify weight sets are removed
ceph osd crush dump | grep weight_set
```

## Summary

Compat weight sets in Ceph CRUSH allow the balancer module to fine-tune OSD weights for better data distribution without modifying the primary CRUSH topology. Enable `crush-compat` mode in the balancer for maximum client compatibility, or use `upmap` mode for more precise per-PG optimization. The balancer manages compat weight sets automatically when running in auto mode, gradually improving distribution over time.
