# How to Monitor Ceph IOPS Metrics

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Monitoring, IOPS, Prometheus, Grafana

Description: Learn how to monitor Ceph IOPS metrics using Prometheus to track read and write operations per second, identify hot pools, and alert on performance anomalies.

---

## Understanding Ceph IOPS Metrics

IOPS (Input/Output Operations Per Second) in Ceph measures the rate of read and write requests processed by the cluster. Key Prometheus metrics include:

- `ceph_osd_op_r` - read operations counter per OSD
- `ceph_osd_op_w` - write operations counter per OSD
- `ceph_pool_rd` - read operations per pool
- `ceph_pool_wr` - write operations per pool

Apply `rate()` to convert these counters to operations per second.

## Enable the Prometheus Exporter

Ensure monitoring is enabled in the Rook CephCluster:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  monitoring:
    enabled: true
    rulesNamespace: rook-ceph
```

Verify the metrics are being scraped:

```bash
kubectl -n rook-ceph get pods | grep mgr
kubectl -n rook-ceph port-forward svc/rook-ceph-mgr 9283:9283 &
curl -s http://localhost:9283/metrics | grep ceph_osd_op_r
```

## Query IOPS in Prometheus

Calculate read and write IOPS at cluster and pool levels:

```bash
# Cluster-level read IOPS (sum across all OSDs)
sum(rate(ceph_osd_op_r[5m]))

# Cluster-level write IOPS
sum(rate(ceph_osd_op_w[5m]))

# Per-pool read IOPS
rate(ceph_pool_rd[5m])

# Per-pool write IOPS
rate(ceph_pool_wr[5m])

# Top 5 pools by total IOPS
topk(5, rate(ceph_pool_rd[5m]) + rate(ceph_pool_wr[5m]))
```

## Check IOPS via CLI

For real-time IOPS without Prometheus:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash -c "
  ceph -s
  watch -n 1 ceph osd pool stats
"
```

The pool stats output shows current IOPS per pool:

```text
pool rbd id 1
  client io 2456 op/s rd, 890 op/s wr
```

## Benchmark IOPS with FIO

Use FIO inside a test pod to measure actual block device IOPS:

```bash
# Create a test RBD PVC, then run fio
kubectl run fio-test --image=ljishen/fio --restart=Never \
  --overrides='{"spec":{"volumes":[{"name":"data","persistentVolumeClaim":{"claimName":"test-pvc"}}],"containers":[{"name":"fio-test","image":"ljishen/fio","volumeMounts":[{"mountPath":"/data","name":"data"}],"command":["fio","--name=randread","--ioengine=libaio","--iodepth=32","--rw=randread","--bs=4k","--direct=1","--size=1G","--filename=/data/test.img","--runtime=60","--time_based"]}]}}'
```

## Set Up IOPS Alerts

Alert on sustained high IOPS that might indicate runaway workloads:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: ceph-iops-alerts
  namespace: rook-ceph
spec:
  groups:
    - name: ceph-iops
      rules:
        - alert: CephHighWriteIOPS
          expr: sum(rate(ceph_osd_op_w[5m])) > 50000
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Ceph cluster write IOPS above 50k for 5 minutes"
            description: "Current write IOPS: {{ $value | humanize }}"
        - alert: CephPoolHighIOPS
          expr: rate(ceph_pool_rd[5m]) + rate(ceph_pool_wr[5m]) > 10000
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "Pool {{ $labels.pool_id }} IOPS above 10k"
```

## Grafana Dashboard Panels

Configure these panels in Grafana for IOPS visibility:

- Time series: cluster read IOPS vs. write IOPS (last 1 hour)
- Time series: per-pool IOPS breakdown (stacked)
- Stat panel: current total cluster IOPS with thresholds
- Bar gauge: top 10 pools by write IOPS

## Summary

Monitoring Ceph IOPS requires using `rate()` in Prometheus on the `ceph_osd_op_r`, `ceph_osd_op_w`, `ceph_pool_rd`, and `ceph_pool_wr` metrics to get operations per second. Combining per-pool visibility with alerting thresholds and FIO benchmarks helps you identify hot pools, size hardware appropriately, and detect runaway workloads before they saturate the cluster.
