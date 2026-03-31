# How to Set and Unset the norecover Flag in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, OSD, Recovery, Flag, Maintenance

Description: Learn how the Ceph norecover flag pauses object recovery and when to use it to protect cluster performance during maintenance or high-load periods.

---

When Ceph detects degraded PGs (PGs with fewer replicas than desired), it begins object recovery to restore full redundancy. The `norecover` flag pauses this recovery process, letting you control when the I/O overhead of recovery occurs.

## What Recovery Is

Recovery happens when an OSD comes back after being `down` and needs its objects updated to reflect writes that occurred while it was absent. It is distinct from backfill, which populates new or empty OSDs with PG data.

Recovery reads objects from the primary OSD, sends them to the lagging OSD, and updates the PG state from `degraded` to `active+clean`.

## Setting norecover

```bash
ceph osd set norecover
```

Check the flag is active:

```bash
ceph osd dump | grep flags
ceph status
# HEALTH_WARN: norecover flag(s) set
```

Note that the cluster will still show `HEALTH_WARN` for degraded PGs - `norecover` just prevents the repair work, not the detection.

## Use Case - Protecting IOPS During Peak Hours

If a disk failure occurs during a critical workload period, you may want to defer recovery:

```bash
# Defer recovery until off-peak hours
ceph osd set norecover

# Confirm the cluster is still serving I/O
ceph status

# During maintenance window
ceph osd unset norecover
```

## Use Case - Preventing Recovery Saturation After Multiple Failures

If several OSDs fail simultaneously, recovery can saturate your network. You can limit the rate before allowing full recovery:

```bash
# Throttle recovery first
ceph config set osd osd_recovery_max_active 1
ceph config set osd osd_recovery_op_priority 1

# Then allow recovery
ceph osd unset norecover
```

## Monitoring Recovery Progress

Once you unset `norecover`, monitor the progress:

```bash
watch ceph status
# Look for: "X objects recovering in Y PGs"

ceph pg stat
# active+degraded count should decrease
```

## Recovery Tuning Parameters

```bash
# Limit concurrent recovery operations per OSD
ceph config set osd osd_recovery_max_active_hdd 1
ceph config set osd osd_recovery_max_active_ssd 3

# Set recovery priority vs client I/O
ceph config set osd osd_recovery_op_priority 3
```

## Unsetting norecover

```bash
ceph osd unset norecover
```

Recovery begins immediately. Confirm the cluster converges to `HEALTH_OK`:

```bash
watch ceph health detail
```

## Difference from nobackfill

- `norecover`: stops recovery of existing PGs that lost replicas
- `nobackfill`: stops migration of PGs to newly added OSDs

Both pause data movement, but they target different scenarios.

## Summary

The `norecover` flag lets you defer object recovery to protect client I/O performance during critical periods. It should be treated as a short-term measure - leaving it set means degraded PGs persist, and any additional OSD failure while recovery is paused could result in data loss if replication falls below the minimum requirement.
