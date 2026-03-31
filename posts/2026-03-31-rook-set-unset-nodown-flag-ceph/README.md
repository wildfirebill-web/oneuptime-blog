# How to Set and Unset the nodown Flag in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, OSD, Flag, Maintenance, Storage

Description: Learn how the Ceph nodown flag prevents monitors from marking OSDs as down and when to use it during cluster maintenance operations.

---

The `nodown` flag instructs Ceph monitors not to mark OSDs as `down`, even if they stop sending heartbeats. This is rarely used in production but can be valuable in specific diagnostic and maintenance scenarios.

## What nodown Does

Normally, if an OSD misses enough heartbeat intervals, the monitor marks it `down`. The OSD transitions to `down+in`, which is a degraded state but does not trigger data movement. With `nodown` set, the monitor will not mark any OSD as down regardless of missed heartbeats.

## Setting the nodown Flag

```bash
ceph osd set nodown
```

Confirm it is active:

```bash
ceph osd dump | grep flags
ceph status
# HEALTH_WARN: nodown flag(s) set
```

## Use Case - Network Flapping

If your cluster is experiencing intermittent network issues that cause OSDs to repeatedly flip between `up` and `down`, this creates excessive OSD map epochs and cluster churn. Setting `nodown` temporarily freezes OSD state while you investigate:

```bash
ceph osd set nodown
# Investigate network issues
ping -c 100 osd-node1
traceroute osd-node1
# Fix network
ceph osd unset nodown
```

## Use Case - Preventing Cascading Alerts

During a maintenance window where you know some OSD heartbeats will be temporarily disrupted (for example, during a switch upgrade), `nodown` prevents a cascade of `HEALTH_WARN` and recovery operations:

```bash
ceph osd set nodown
# Perform switch upgrade - some heartbeats may be lost
# After switch is back
ceph osd unset nodown
```

## Checking OSD Heartbeat Status

Even with `nodown` set, you can still inspect OSD connectivity:

```bash
ceph osd stat
ceph osd dump | grep -E "osd\.[0-9]+ (up|down)"
```

## Manually Marking OSDs Down

While `nodown` is set, you can still manually force an OSD to the `down` state:

```bash
ceph osd down osd.5
```

This is useful for testing failover behavior without waiting for the monitor to detect a real failure.

## Unsetting nodown

```bash
ceph osd unset nodown
```

After removing the flag, the monitors resume normal heartbeat monitoring. Any OSDs that are genuinely unreachable will quickly be marked `down`.

```bash
ceph health
watch ceph osd stat
```

## Risk Warning

Leaving `nodown` set for extended periods is dangerous. If OSDs genuinely fail while the flag is set, Ceph will not detect the failure, PGs will not be remapped, and client I/O to affected PGs will hang or fail. Always remove the flag as soon as possible.

## Summary

The `nodown` flag prevents monitors from marking OSDs down due to missed heartbeats. It is a short-term tool for managing network instability or maintenance windows where brief connectivity disruptions are expected. Use it cautiously and always remove it immediately after the maintenance window closes.
