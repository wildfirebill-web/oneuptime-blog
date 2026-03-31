# How to Monitor Ceph Client IO Patterns in Grafana

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Grafana, IO, Client, Monitoring, Performance, Dashboard

Description: Monitor Ceph client read/write IO patterns, IOPS, throughput, and latency in Grafana to understand workload behavior and optimize storage performance.

---

## Understanding Client IO Patterns

Different workloads generate very different IO patterns. Databases produce small random reads and writes, while video streaming generates large sequential reads. Understanding your client IO pattern is essential for tuning Ceph pool settings like `rbd_cache`, `osd_op_threads`, and erasure coding parameters.

## Key Client IO Metrics

The Ceph manager exposes detailed per-OSD and aggregate IO metrics:

```promql
# Total client read operations per second
rate(ceph_osd_op_r{namespace="rook-ceph"}[5m])

# Total client write operations per second
rate(ceph_osd_op_w{namespace="rook-ceph"}[5m])

# Read bytes per second
rate(ceph_osd_op_r_out_bytes{namespace="rook-ceph"}[5m])

# Write bytes per second
rate(ceph_osd_op_w_in_bytes{namespace="rook-ceph"}[5m])
```

## IOPS Dashboard Panel

Create a Time series panel with read and write IOPS:

```promql
# Aggregate read IOPS
sum(rate(ceph_osd_op_r{namespace="rook-ceph"}[5m]))

# Aggregate write IOPS
sum(rate(ceph_osd_op_w{namespace="rook-ceph"}[5m])) * -1
```

Multiply writes by -1 to display them below the x-axis, creating a classic "butterfly" IO chart that makes read/write balance immediately visible.

## IO Size Distribution

Calculate average IO size to distinguish random vs. sequential workloads:

```promql
# Average read IO size in KB
(
  rate(ceph_osd_op_r_out_bytes{namespace="rook-ceph"}[5m]) /
  rate(ceph_osd_op_r{namespace="rook-ceph"}[5m])
) / 1024

# Average write IO size in KB
(
  rate(ceph_osd_op_w_in_bytes{namespace="rook-ceph"}[5m]) /
  rate(ceph_osd_op_w{namespace="rook-ceph"}[5m])
) / 1024
```

Small average IO sizes (under 64 KB) indicate random workloads; large sizes (over 512 KB) indicate sequential.

## Client Latency Panels

Track read and write latency separately:

```promql
# Average read latency in milliseconds
(
  rate(ceph_osd_op_r_latency_sum{namespace="rook-ceph"}[5m]) /
  rate(ceph_osd_op_r_latency_count{namespace="rook-ceph"}[5m])
) * 1000

# Average write latency in milliseconds
(
  rate(ceph_osd_op_w_latency_sum{namespace="rook-ceph"}[5m]) /
  rate(ceph_osd_op_w_latency_count{namespace="rook-ceph"}[5m])
) * 1000
```

## Identifying Hot OSDs

Find OSDs handling disproportionate load:

```promql
# Per-OSD IOPS
rate(ceph_osd_op{namespace="rook-ceph"}[5m])
```

Use a Bar chart grouped by `ceph_daemon` to spot hot spots. Uneven distribution may indicate CRUSH map issues.

## IO Pattern Over Time

Use a Heatmap panel to visualize IO pattern changes throughout the day:

```promql
# IOPS by hour (rate over 1m intervals for fine granularity)
rate(ceph_osd_op{namespace="rook-ceph"}[1m])
```

This reveals diurnal patterns like nightly backup spikes or business-hours peak load.

## Summary

Monitoring Ceph client IO patterns in Grafana reveals whether workloads are read- or write-heavy, random or sequential, and whether load is balanced across OSDs. The butterfly IOPS chart, average IO size calculation, and per-OSD heatmaps give you the data needed to tune Ceph configuration, resize pools, or add OSDs to relieve hot spots before they cause latency problems.
