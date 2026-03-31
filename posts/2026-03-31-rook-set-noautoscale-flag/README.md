# How to Set the Noautoscale Flag in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Configuration, Placement Group, Storage

Description: Learn how to use the Ceph noautoscale flag to globally disable PG autoscaling across all pools for stable, predictable cluster behavior.

---

## Understanding the noautoscale Flag

Ceph's PG autoscaler automatically adjusts placement group counts for pools based on utilization and the `mon_target_pg_per_osd` target. While useful for dynamic workloads, there are scenarios where you want to freeze PG counts globally - for example, during major maintenance, cluster migrations, or when autoscaling is causing unwanted rebalancing.

The `noautoscale` flag is an OSD flag that disables the autoscaler across all pools simultaneously, regardless of each pool's individual `pg_autoscale_mode` setting.

## Checking Autoscale Status

```bash
# Check if noautoscale is currently set
ceph osd dump | grep noautoscale

# View per-pool autoscale modes
ceph osd pool autoscale-status

# Check active OSD flags
ceph osd dump | grep flags
```

Example output when noautoscale is not set:

```text
flags sortbitwise,recovery_deletes,purged_snapdirs,pglog_hardlimit
```

## Setting the noautoscale Flag

```bash
# Disable autoscaling globally
ceph osd set noautoscale

# Verify the flag is active
ceph osd dump | grep flags
# Output: flags sortbitwise,recovery_deletes,...,noautoscale
```

After setting this flag, the autoscaler will not change any pool's PG count even if it detects imbalances.

## Removing the noautoscale Flag

```bash
# Re-enable global autoscaling
ceph osd unset noautoscale

# Verify the flag is removed
ceph osd dump | grep flags
```

## Per-Pool vs Global Control

You can also control autoscaling at the pool level without using the global flag:

```bash
# Disable autoscaling on a specific pool
ceph osd pool set mypool pg_autoscale_mode off

# Set to warn mode (report recommendations without changing PG count)
ceph osd pool set mypool pg_autoscale_mode warn

# Enable autoscaling on a specific pool
ceph osd pool set mypool pg_autoscale_mode on

# Check all pools' autoscale modes
ceph osd pool autoscale-status
```

## When to Use noautoscale

Use the global `noautoscale` flag in these scenarios:

```text
1. Before performing OSD replacement or cluster expansion
   - Prevents PG splitting during hardware changes

2. During controlled maintenance windows
   - Keeps PG distribution stable while making config changes

3. When investigating autoscaler misbehavior
   - Freezes the current state for troubleshooting

4. In compliance environments requiring predictable storage topology
   - Prevents unexpected rebalancing events
```

## Temporarily Disabling Autoscale for Maintenance

```bash
#!/bin/bash
# Maintenance window script

# Disable autoscaling before maintenance
ceph osd set noautoscale
echo "Autoscaling disabled"

# Perform your maintenance here
# ...

# Re-enable autoscaling after maintenance
ceph osd unset noautoscale
echo "Autoscaling re-enabled"

# Check if any autoscale recommendations are pending
ceph osd pool autoscale-status
```

## Interaction with Other OSD Flags

The `noautoscale` flag works alongside other OSD flags:

```bash
# Common flags you might set during maintenance
ceph osd set noout          # prevent OSDs from being marked out
ceph osd set norebalance    # stop rebalancing data
ceph osd set noautoscale    # stop PG count changes

# Clear all three after maintenance
ceph osd unset noout
ceph osd unset norebalance
ceph osd unset noautoscale
```

## Summary

The `noautoscale` OSD flag is a simple but powerful tool to globally freeze Ceph's PG autoscaler. Set it with `ceph osd set noautoscale` before planned maintenance or cluster changes, and clear it with `ceph osd unset noautoscale` afterward. For finer control, use per-pool `pg_autoscale_mode` settings instead of the global flag.
