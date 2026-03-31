# How to Monitor NVMe-oF Gateway Performance in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, NVMe-oF, Monitoring, Performance

Description: Monitor Ceph NVMe-oF gateway performance using ceph CLI commands, Prometheus metrics, and kernel-level tools to identify bottlenecks and ensure SLA compliance.

---

## Overview

NVMe-oF gateway performance monitoring covers multiple layers: Ceph-level IOPS and throughput, gateway process CPU/memory, kernel NVMe statistics on initiators, and network utilization. This guide covers all layers.

## Monitor Gateway Status with Ceph CLI

```bash
# Gateway health
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  ceph nvmeof gateway info

# List subsystems and namespaces
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  ceph nvmeof subsystem list

# Check namespace I/O counters
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  ceph nvmeof namespace list --nqn nqn.2024-01.io.ceph:mysubsystem
```

## Monitor at the OSD Level

NVMe-oF traffic goes through regular Ceph RADOS operations. Monitor at the pool level:

```bash
# Pool I/O statistics
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  ceph osd pool stats nvmeof-pool

# Real-time I/O monitoring
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  ceph iostat 2

# OSD performance dump
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  ceph osd perf
```

## Prometheus Metrics for NVMe-oF

Rook exposes Ceph metrics via the Prometheus endpoint. Key metrics to watch:

```bash
# Query gateway-related metrics
curl -s http://rook-ceph-mgr.rook-ceph.svc:9283/metrics | grep nvmeof

# Key metrics:
# ceph_nvmeof_gateway_state - gateway health (1=up, 0=down)
# ceph_pool_rd_bytes - read bytes for nvmeof pool
# ceph_pool_wr_bytes - write bytes for nvmeof pool
# ceph_pool_rd - read IOPS for nvmeof pool
# ceph_pool_wr - write IOPS for nvmeof pool
```

Create a Prometheus alert for gateway downtime:

```yaml
groups:
  - name: nvmeof
    rules:
      - alert: NVMeoFGatewayDown
        expr: ceph_nvmeof_gateway_state == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "NVMe-oF gateway is down"
          description: "Gateway {{ $labels.gateway }} has been down for 2 minutes"
```

## Monitor on Initiator Node

On initiator nodes, use nvme-cli to gather I/O statistics:

```bash
# Get SMART/health data and I/O stats
nvme smart-log /dev/nvme0n1

# Get I/O completion statistics
nvme get-log /dev/nvme0n1 --log-id=0x02

# Continuous monitoring with iostat
iostat -x /dev/nvme0n1 2

# Monitor latency distribution
nvme latency-stats /dev/nvme0n1
```

## Grafana Dashboard

Import the Rook-Ceph Grafana dashboard and filter by `nvmeof-pool`:

```bash
# Port-forward Grafana
kubectl -n rook-ceph port-forward svc/rook-ceph-mgr-dashboard 8443:8443

# Import dashboard ID 2842 (Ceph - OSD (Single)) and filter by pool
```

Key metrics to graph:
- Pool read/write IOPS
- Pool read/write latency (p99)
- Gateway CPU utilization
- Network bandwidth per gateway pod

## Summary

Monitoring Ceph NVMe-oF gateway performance requires observing multiple layers: ceph CLI for gateway and pool stats, Prometheus alerts for gateway availability, iostat and nvme-cli on initiator nodes for per-device metrics, and Grafana for visualization. Setting latency SLO alerts on the nvmeof pool ensures early warning before performance degrades to impact applications.
