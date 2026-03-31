# How to Balance QoS Between Recovery and Client IO

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, QoS, Recovery, Performance

Description: Learn how to balance Ceph OSD resources between client I/O and background recovery operations to minimize performance impact during healing.

---

## The Recovery vs Client IO Tradeoff

When Ceph recovers after OSD failures or rebalancing, background recovery I/O competes with client workloads for the same OSD resources. Without limits, recovery can saturate OSD throughput and cause severe latency spikes for applications.

The mClock scheduler provides tools to enforce a controlled balance.

## Understanding the Three I/O Classes

Ceph categorizes OSD operations into three classes:

1. **Client ops** - Direct application I/O (reads, writes)
2. **Background recovery** - Replication repair after OSD failures
3. **Background best-effort** - Scrubbing, backfill, balance

Each class has independent reservation, weight, and limit parameters.

## Using Built-In Profiles

The fastest way to adjust the balance is using mClock profiles:

```bash
# Prioritize client I/O - recovery is slowed
ceph config set osd osd_mclock_profile high_client_ops

# Balance client and recovery equally
ceph config set osd osd_mclock_profile balanced

# Prioritize recovery - good after major failures
ceph config set osd osd_mclock_profile high_recovery_ops
```

Switch profiles based on current operational needs:

```bash
# Check current profile
ceph config get osd osd_mclock_profile

# Switch to balanced during normal operations
ceph config set osd osd_mclock_profile balanced
```

## Custom Recovery Limits

Fine-tune recovery bandwidth after a large OSD failure to minimize client impact:

```bash
# Limit recovery to 50 IOPS per OSD
ceph config set osd osd_mclock_scheduler_background_recovery_lim 50

# Give client ops much higher weight
ceph config set osd osd_mclock_scheduler_client_wgt 500
ceph config set osd osd_mclock_scheduler_background_recovery_wgt 50
```

## Controlling OSD-Level Recovery Throttles

Additional recovery throttles beyond mClock:

```bash
# Limit bytes per second sent during recovery
ceph config set osd osd_recovery_max_active 3
ceph config set osd osd_recovery_sleep 0.1
ceph config set osd osd_recovery_op_priority 3
```

View recovery progress:

```bash
ceph status | grep -A5 recovery
ceph pg stat
```

## Temporarily Boosting Recovery Speed

After a major failure, you may want to accelerate recovery before re-applying limits:

```bash
# Temporarily switch to high recovery profile
ceph config set osd osd_mclock_profile high_recovery_ops

# Boost recovery operation limits
ceph config set osd osd_recovery_max_active 8
ceph config set osd osd_recovery_sleep 0

# Monitor progress
watch ceph pg stat
```

Once recovery completes, restore normal settings:

```bash
ceph config set osd osd_mclock_profile high_client_ops
ceph config set osd osd_recovery_max_active 3
ceph config set osd osd_recovery_sleep 0.1
```

## Measuring Client Impact During Recovery

Benchmark client latency with and without recovery happening:

```bash
# Start recovery (set one OSD out and then back in)
ceph osd out osd.0 && ceph osd in osd.0

# Measure client latency during recovery
rados bench -p prodpool 60 write -t 4 | grep -E "avg lat|IOPS"

# Compare to baseline without recovery
rados bench -p prodpool 60 write -t 4 | grep -E "avg lat|IOPS"
```

## Automation with Scripts

Automatically switch profiles based on recovery activity:

```bash
#!/bin/bash
RECOVERING=$(ceph pg stat --format json 2>/dev/null | \
  python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('num_pgs_active_recovery', 0))")

if [ "$RECOVERING" -gt 100 ]; then
  ceph config set osd osd_mclock_profile high_client_ops
  echo "$(date): Recovery active ($RECOVERING PGs), using high_client_ops profile"
else
  ceph config set osd osd_mclock_profile balanced
  echo "$(date): Normal operations, using balanced profile"
fi
```

## Summary

Balancing QoS between recovery and client I/O in Ceph is best managed through the mClock profile system, with manual tuning of recovery limits when more precise control is needed. Switching to `high_client_ops` during normal operations protects application performance, while temporarily enabling `high_recovery_ops` after major failures accelerates healing. Automated profile switching based on recovery PG counts provides a hands-off approach to dynamic QoS management.
