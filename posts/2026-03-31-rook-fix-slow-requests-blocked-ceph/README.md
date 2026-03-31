# How to Fix 'slow requests are blocked' in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Troubleshooting, Performance, Slow Request, OSD, Latency

Description: Diagnose and fix slow and blocked requests in Ceph by identifying disk bottlenecks, network issues, and OSD thread starvation.

---

## Introduction

Ceph reports "slow requests" when OSD operations take longer than `osd_op_complaint_time` (default 30 seconds). This manifests as high I/O latency for clients and can escalate to blocked requests that prevent progress entirely. This guide covers systematic diagnosis and resolution.

## Identifying Slow Requests

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph health detail
```

Example:

```text
HEALTH_WARN 8 slow requests are blocked for more than 32 sec
8 slow requests are blocked
```

Get OSD-level details:

```bash
ceph osd perf
```

Output shows latency per OSD - look for outliers with high `commit_latency_ms`.

## Step 1 - Identify the Slow OSD

```bash
# Check per-OSD operation queue
kubectl -n rook-ceph exec -it rook-ceph-osd-0-<pod-id> -- \
  ceph daemon osd.0 dump_ops_in_flight
```

Look for operations that have been in flight for many seconds.

## Step 2 - Check Disk Performance

High disk latency is the most common cause:

```bash
# Run on the node hosting the slow OSD
iostat -x 1 10 | grep sdb
```

High `await` values (>10ms for SSDs, >20ms for spinning disks) indicate disk problems.

Test raw disk performance:

```bash
fio --name=test --rw=randwrite --bs=4k --numjobs=4 \
  --iodepth=32 --direct=1 --filename=/dev/sdb \
  --size=1G --runtime=30 --time_based
```

## Step 3 - Check OSD Memory

OSDs with insufficient memory thrash their cache:

```bash
ceph config get osd osd_memory_target
# Default: 4GB
# Increase if nodes have RAM available:
ceph config set osd osd_memory_target 8589934592  # 8GB
```

## Step 4 - Check Network Latency

High network latency between OSDs causes slow replication:

```bash
# Check OSD network stats
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd perf

# Ping between OSD nodes
kubectl -n rook-ceph exec -it rook-ceph-osd-0-<pod-id> -- \
  ping -c 10 <osd-1-pod-ip>
```

## Step 5 - Reduce Recovery Traffic

Ongoing recovery can starve foreground I/O:

```bash
# Limit recovery operations
ceph config set osd osd_max_backfills 1
ceph config set osd osd_recovery_max_active 1
ceph config set osd osd_recovery_sleep 0.05
```

## Step 6 - Check for Throttled OSDs

```bash
ceph osd tree | grep -E "down|out"
```

If OSDs are flapping, the cluster continuously tries to recover, causing slow requests.

## Monitoring Resolution

```bash
watch "kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph health"
```

The slow request count should decrease as the root cause is addressed.

## Summary

Slow and blocked requests in Ceph usually stem from disk latency, insufficient OSD memory, network congestion between OSDs, or recovery traffic competing with foreground I/O. The diagnostic approach involves checking OSD-level operation queues, disk performance metrics, and network latency, then applying targeted configuration changes to reduce pressure on the bottlenecked resource.
