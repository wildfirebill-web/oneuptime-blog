# How to Let the Balancer Module Manage CRUSH Weights Automatically

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Balancer, CRUSH, Storage

Description: Learn how to enable and configure the Ceph balancer module to automatically optimize CRUSH weights and PG distribution for even data placement.

---

## What the Balancer Module Does

The Ceph balancer mgr module continuously evaluates data distribution across OSDs and automatically adjusts placement to minimize deviation from ideal balance. It operates in two modes:

- **crush-compat** - modifies CRUSH weight sets to influence PG placement; compatible with old clients
- **upmap** - creates per-PG upmap entries that override CRUSH decisions; more precise but requires newer clients

Without the balancer, CRUSH weight imbalances (from differently-sized drives or removal of OSDs) can leave some OSDs significantly more loaded than others.

## Enabling the Balancer

```bash
# Enable the balancer mgr module
ceph mgr module enable balancer

# Check available modes
ceph balancer mode

# Set mode and turn on
ceph balancer mode upmap
ceph balancer on

# Verify status
ceph balancer status
```

## Choosing the Right Mode

```bash
# Use upmap for precise per-PG control (recommended for new clusters)
ceph balancer mode upmap

# Use crush-compat for compatibility with older clients (pre-Luminous)
ceph balancer mode crush-compat

# Check minimum client version in the cluster
ceph features
```

`upmap` mode is preferred for Luminous and later because it is more accurate and does not modify the CRUSH map structure.

## Evaluating Current Balance

```bash
# Get a score for current distribution (lower is better)
ceph balancer eval

# Example output:
# current score 0.024 (0 pools)

# Evaluate distribution per pool
ceph balancer eval mypool

# Show verbose distribution breakdown
ceph osd df | tail -5
```

## Running a Manual Optimization Pass

```bash
# Create an optimization plan without executing
ceph balancer optimize myplan

# Review the plan
ceph balancer show myplan

# Execute the plan after review
ceph balancer execute myplan

# Check new score
ceph balancer eval
```

## Configuring Automatic Balancing

When `ceph balancer on` is set, the balancer runs automatically on a schedule:

```bash
# Set how aggressively the balancer runs (0-1, default 0.5)
ceph config set mgr mgr/balancer/sleep_interval 60

# Set max PG moves per balancer cycle
ceph config set mgr mgr/balancer/max_misplaced 0.05

# Check current auto-balance settings
ceph config get mgr mgr/balancer/sleep_interval
```

## Controlling Which Pools are Balanced

```bash
# By default, all pools are balanced
# Exclude a pool from balancing
ceph osd pool set mypool nopgchange true

# Check which pools have balancing excluded
ceph osd dump | grep nopgchange

# Explicitly include a pool
ceph osd pool set mypool nopgchange false
```

## Monitoring Balancer Activity

```bash
# Watch balancer progress
watch -n 5 ceph balancer status

# Monitor data migration from balancer
ceph -s | grep misplaced

# Check OSD utilization after balancing
ceph osd df | sort -k 6 -n | tail -10
```

## Turning Off the Balancer

```bash
# Pause automatic balancing (keeps module loaded)
ceph balancer off

# Completely disable the module
ceph mgr module disable balancer
```

## Summary

The Ceph balancer module automates CRUSH weight management and PG distribution optimization. Enable it with `ceph balancer on` after selecting `upmap` mode for modern clusters or `crush-compat` for older client compatibility. The balancer runs continuously in the background, evaluating distribution quality and applying incremental optimizations to keep OSD utilization balanced without manual CRUSH map adjustments.
