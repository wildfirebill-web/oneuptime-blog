# How to Enable Per-Image IO Statistics for RBD in Rook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RBD, Monitoring, Performance

Description: Enable per-image RBD IO statistics in Rook-Ceph to get granular IOPS, throughput, and latency metrics for individual RBD volumes.

---

## Overview

By default, Ceph tracks IO statistics at the pool level. Enabling per-image statistics allows you to see metrics for individual RBD images (volumes), which is critical for diagnosing which specific PVC is causing performance issues in a multi-tenant cluster.

## Enable RBD Statistics Collection

Enable the `rbd_stats_pools` configuration option to tell Ceph which pools to collect per-image statistics for:

```bash
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph config set mgr mgr/prometheus/rbd_stats_pools "replicapool"
```

For multiple pools:

```bash
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph config set mgr mgr/prometheus/rbd_stats_pools "replicapool,fast-pool,ec-pool"
```

Use `*` to collect from all pools (may be resource-intensive on large clusters):

```bash
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph config set mgr mgr/prometheus/rbd_stats_pools "*"
```

## Configure the Scrape Interval

Set how frequently the MGR collects RBD statistics:

```bash
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph config set mgr mgr/prometheus/rbd_stats_pools_refresh_interval 60
```

## Verify the Configuration

Check that the setting is applied:

```bash
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph config get mgr mgr/prometheus/rbd_stats_pools
```

Restart the MGR to apply the change:

```bash
kubectl rollout restart deployment -n rook-ceph -l app=rook-ceph-mgr
```

## Query Per-Image Metrics in Prometheus

After enabling, per-image metrics appear in Prometheus with the `ceph_rbd_` prefix:

```bash
kubectl port-forward -n monitoring svc/prometheus-operated 9090:9090 &

# IOPS per RBD image
curl -s 'http://localhost:9090/api/v1/query?query=rate(ceph_rbd_write_ops[5m])' | jq .

# Write throughput per image
curl -s 'http://localhost:9090/api/v1/query?query=rate(ceph_rbd_write_bytes[5m])' | jq .
```

Key per-image metrics available:

```text
ceph_rbd_read_bytes           - Read bytes per image
ceph_rbd_write_bytes          - Write bytes per image
ceph_rbd_read_ops             - Read operations per image
ceph_rbd_write_ops            - Write operations per image
ceph_rbd_read_latency_sum     - Cumulative read latency
ceph_rbd_write_latency_sum    - Cumulative write latency
```

## Find the Kubernetes PVC for an RBD Image

The `image` label in the metrics corresponds to the RBD image name. Map it to a Kubernetes PVC:

```bash
# Find PV with the RBD image name
kubectl get pv -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.csi.volumeAttributes.imageName}{"\n"}{end}' | grep <image-name>
```

## Create a Grafana Dashboard Variable for RBD Images

In a Grafana dashboard, add a variable to filter by RBD image:

```text
Type: Query
Query: label_values(ceph_rbd_write_ops, image)
Label: RBD Image
```

## Summary

Enabling per-image RBD IO statistics with `rbd_stats_pools` configuration gives operators granular visibility into the storage consumption and performance of individual Kubernetes PVCs. This is invaluable in multi-tenant clusters where a noisy workload can be identified by its specific volume name and traced back to the consuming pod.
