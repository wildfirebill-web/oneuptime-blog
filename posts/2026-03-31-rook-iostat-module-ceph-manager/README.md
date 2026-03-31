# How to Use the Iostat Module in Ceph Manager

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, IO Statistics, Performance, Monitoring

Description: Learn how to use the Ceph Manager iostat module to monitor real-time I/O statistics for OSDs and pools in your Ceph cluster.

---

The Ceph Manager iostat module provides a real-time view of I/O statistics similar to the Linux `iostat` command, but specifically for Ceph OSDs and storage pools. It helps operators quickly identify performance bottlenecks and understand workload distribution.

## Enabling the Module

The iostat module is part of the Ceph Manager built-in modules. Enable it with:

```bash
ceph mgr module enable iostat
```

## Running the Iostat Command

Once enabled, use the `ceph iostat` command to get a live dashboard of I/O metrics:

```bash
ceph iostat
```

This streams output in a format similar to `iostat -x`, refreshing every second:

```text
2026-03-31 10:00:00
                   Read       Write      Read IOPS  Write IOPS
  Client I/O:    128 MiB/s   45 MiB/s    512        180
  Recovery I/O:    0 B/s      0 B/s       0          0
  Backfill I/O:    0 B/s      0 B/s       0          0
```

## Setting the Refresh Interval

Change the refresh rate in seconds:

```bash
ceph iostat -p 2
```

This refreshes every 2 seconds instead of the default 1 second.

## Per-Pool Statistics

To see per-pool breakdown, query the pool performance counters:

```bash
ceph osd pool stats
```

Example output:

```text
pool production id 1
  client io 65 MiB/s rd, 12 MiB/s wr, 260 op/s rd, 48 op/s wr

pool backups id 2
  nothing is going on
```

## OSD-Level I/O Stats

For deeper per-OSD metrics, use the performance dump:

```bash
ceph osd perf
```

```text
osd.0  apply_latency_ms  commit_latency_ms
        1.23              2.45
osd.1   0.98              1.87
```

## Integrating with Monitoring

The iostat data is also available via the Prometheus module. Relevant metrics include:

```bash
# Query Prometheus for OSD read IOPS
curl -s 'http://192.168.1.10:9283/metrics' | grep ceph_osd_op_r
```

## Detecting Hotspots

Use iostat combined with pool stats to find hotspot OSDs:

```bash
# Watch for OSDs with unusually high commit latency
watch -n 2 "ceph osd perf | sort -k3 -rn | head -10"
```

High commit latency on specific OSDs often indicates slow disks or network issues on those nodes.

## Summary

The Ceph Manager iostat module provides real-time cluster-wide I/O metrics via the `ceph iostat` command, displaying read/write throughput and IOPS for client, recovery, and backfill traffic. For detailed analysis, combining iostat output with `ceph osd perf` and per-pool statistics helps operators quickly pinpoint I/O bottlenecks across the storage cluster.
