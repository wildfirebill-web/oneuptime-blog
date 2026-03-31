# How to Monitor Ceph Disk IO Wait Times

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Monitoring, Disk IO, Performance

Description: Learn how to monitor Ceph disk IO wait times at both the OS and Ceph levels to detect slow disks, identify OSD bottlenecks, and correlate disk latency with Ceph performance.

---

## Why Disk IO Wait Matters

IO wait time measures how long CPU cores sit idle waiting for disk operations to complete. Elevated IO wait on Ceph OSD nodes directly translates to higher OSD latency, degraded cluster performance, and potential OSD timeouts. Monitoring IO wait times enables you to catch failing or undersized disks before they cause cluster health alerts.

## Monitor IO Wait at the Node Level

Use the Prometheus node exporter to track disk IO wait:

```bash
# IO wait time per disk device
rate(node_disk_io_time_weighted_seconds_total{job="node-exporter"}[5m])

# Average IO wait across all disks on OSD nodes
avg by (instance) (rate(node_disk_io_time_weighted_seconds_total[5m]))

# Disk utilization percentage
rate(node_disk_io_time_seconds_total[5m]) * 100
```

## Check IO Wait with iostat

SSH or exec into an OSD node and run iostat for real-time disk statistics:

```bash
# On the OSD node (via kubectl debug or SSH)
iostat -x 1 10

# Output includes:
# Device  r/s  w/s  rMB/s  wMB/s  await  svctm  %util
# sdb     0.5  500   0.01  250.0   35.2   0.8   40.2
```

The `await` column (average wait time in ms) and `%util` (utilization percentage) are the key indicators. An `await` above 20ms for SSDs or 50ms for HDDs suggests disk saturation.

## Correlate Disk Wait with OSD Latency

Cross-reference node disk IO wait with Ceph OSD latency metrics in Prometheus:

```bash
# Grafana: overlay these two queries
# Line 1: OSD apply latency
ceph_osd_apply_latency_ms

# Line 2: Disk IO wait on the OSD node
rate(node_disk_io_time_weighted_seconds_total{instance="worker-node-1:9100"}[5m]) * 1000
```

When both metrics spike together, the disk is the bottleneck. When OSD latency is high but disk wait is normal, suspect network or CPU issues.

## Check Disk Health with SMART

Disk IO wait spikes may indicate early disk failure. Check SMART health:

```bash
# Run on the OSD node
smartctl -a /dev/sdb

# Key fields to check:
# Reallocated_Sector_Ct - should be 0
# Current_Pending_Sector - should be 0
# Offline_Uncorrectable - should be 0
# UDMA_CRC_Error_Count - high values indicate cable issues
```

## Set Up Disk IO Wait Alerts

Alert on sustained high disk IO wait on OSD nodes:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: ceph-disk-io-alerts
  namespace: rook-ceph
spec:
  groups:
    - name: ceph-disk-io
      rules:
        - alert: CephOSDNodeHighDiskWait
          expr: |
            rate(node_disk_io_time_weighted_seconds_total{instance=~"worker-node.*"}[5m]) > 0.5
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "High disk IO wait on {{ $labels.instance }}, device {{ $labels.device }}"
            description: "IO wait above threshold for 10 minutes - check disk health"
```

## Tune IO Schedulers for OSD Disks

Set the appropriate IO scheduler for OSD disks to minimize wait times:

```bash
# For NVMe SSDs - use 'none' (no scheduler overhead)
echo "none" > /sys/block/nvme0n1/queue/scheduler

# For SATA SSDs - use 'mq-deadline'
echo "mq-deadline" > /sys/block/sda/queue/scheduler

# For HDDs - use 'mq-deadline' or 'bfq'
echo "mq-deadline" > /sys/block/sdb/queue/scheduler
```

Persist these settings via udev rules to survive reboots.

## Summary

Monitoring Ceph disk IO wait combines Prometheus node exporter metrics for trend analysis, `iostat` for real-time per-disk await and utilization, and SMART data for hardware health assessment. Correlating disk wait with OSD latency metrics in Grafana makes it straightforward to distinguish between disk, network, and CPU bottlenecks, enabling targeted remediation.
