# How to Fix DEVICE_HEALTH_TOOMANY Health Check in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, Device, Health Check, OSD

Description: Learn how to fix the DEVICE_HEALTH_TOOMANY warning in Ceph, which fires when too many OSDs are flagged as failing to allow safe simultaneous removal.

---

## What Is DEVICE_HEALTH_TOOMANY?

`DEVICE_HEALTH_TOOMANY` is a Ceph health alert that occurs when the number of OSD devices flagged as unhealthy (by the `devicehealth` module's SMART analysis) exceeds a safe threshold relative to the cluster's replication factor or erasure coding configuration. In other words, Ceph has detected that too many drives are failing at once - and automatically removing all of them would violate the cluster's durability guarantees.

This is a protective measure. If you have a 3-replica pool and 2 OSDs on the same failure domain both fail simultaneously, marking both out could leave PGs with insufficient replicas.

## Checking Cluster State

Start with health detail:

```bash
ceph health detail
```

Typical output:

```text
[ERR] DEVICE_HEALTH_TOOMANY: Too many unhealthy devices - automatic removal paused
    4 out of 12 OSDs are predicted to fail
```

List all flagged devices:

```bash
ceph device ls
ceph device get-health-metrics <device-id>
```

## Why This Is Dangerous

If `min_size` replicas would be violated by removing too many OSDs, PGs become unavailable. Ceph intentionally pauses automatic removal to protect data integrity.

Check your pool's replication settings:

```bash
ceph osd pool get <pool-name> size
ceph osd pool get <pool-name> min_size
```

## Fix Strategy - Remove OSDs One at a Time

You must manually manage the removal order, ensuring the cluster is healthy between each removal.

### Step 1 - Mark One OSD Out

```bash
ceph osd out osd.1
```

### Step 2 - Wait for Rebalance

```bash
watch ceph -s
```

Wait until output shows `HEALTH_OK` or all PGs are `active+clean`.

### Step 3 - Remove the OSD

```bash
ceph osd down osd.1
ceph osd rm osd.1
ceph osd crush remove osd.1
ceph auth del osd.1
```

### Step 4 - Repeat for Next OSD

Only proceed to the next OSD after the cluster has fully rebalanced. This process ensures data durability is maintained throughout.

```bash
ceph osd out osd.4
# wait ...
ceph osd rm osd.4
ceph osd crush remove osd.4
ceph auth del osd.4
```

## Temporarily Disabling Automatic Self-Heal

If you need to pause the `devicehealth` automatic removal while you manage the process manually:

```bash
ceph config set mgr mgr/devicehealth/self_heal false
```

Re-enable after replacements are complete:

```bash
ceph config set mgr mgr/devicehealth/self_heal true
```

## Adjusting the Too-Many Threshold

You can tune how many devices the module considers "too many" before pausing:

```bash
ceph config set mgr mgr/devicehealth/max_devices_failed_per_pool 1
```

## Monitoring

Track device health metrics in Grafana using the Ceph dashboard or Prometheus:

```bash
ceph dashboard check-health
ceph mgr module enable dashboard
```

## Summary

`DEVICE_HEALTH_TOOMANY` is a safety mechanism that pauses automatic OSD removal when simultaneous removals would risk data loss. Fix it by manually removing failing OSDs one at a time, waiting for full rebalance between each removal, and optionally disabling `self_heal` temporarily to take manual control. Always prioritize cluster `HEALTH_OK` state between each removal step.
