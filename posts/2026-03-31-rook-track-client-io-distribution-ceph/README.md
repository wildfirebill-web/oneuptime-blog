# How to Track Client IO Distribution in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Monitoring, Client IO, Performance

Description: Learn how to track client IO distribution in Ceph to identify which clients, pools, or namespaces are consuming the most IOPS and bandwidth for workload management.

---

## Why Track Client IO Distribution

In a shared Ceph cluster, multiple applications and teams compete for IO resources. Understanding which clients generate the most read/write operations helps with capacity planning, QoS enforcement, and troubleshooting noisy-neighbor problems.

## View Current Client IO

Use the Ceph status command for an immediate overview:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash -c "
  ceph -s
"
```

The IO section shows aggregate client read and write rates:

```
  io:
    client:   read: 250 MiB/s, 3200 op/s
              write: 120 MiB/s, 1800 op/s
```

## Per-Pool IO Distribution

Break down IO by pool to identify the busiest pools:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash -c "
  ceph osd pool stats
"
```

Sample output:

```
pool rbd id 1
  client io 2100 op/s rd, 800 op/s wr, 180 MiB/s rd, 90 MiB/s wr

pool cephfs-data id 2
  client io 540 op/s rd, 230 op/s wr, 60 MiB/s rd, 30 MiB/s wr
```

## Prometheus Metrics for Per-Client Tracking

For Kubernetes workloads, correlate PVC names with Ceph pool activity. Use the Rook CSI metrics exposed through csi-metrics-rbdplugin:

```bash
# List CSI metrics endpoints
kubectl -n rook-ceph get pods -l app=csi-rbdplugin -o wide

# Port-forward and inspect metrics
kubectl -n rook-ceph port-forward <csi-pod> 9091:9091 &
curl -s http://localhost:9091/metrics | grep kubelet_volume_stats
```

Key metrics to track per PVC:

```
kubelet_volume_stats_used_bytes{persistentvolumeclaim="my-pvc"}
kubelet_volume_stats_capacity_bytes{persistentvolumeclaim="my-pvc"}
```

## Track IO Per OSD

Identify uneven IO distribution across OSDs, which may indicate CRUSH imbalance:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash -c "
  ceph osd perf
"
```

Compare `op_r` and `op_w` counters across OSDs. Significant differences indicate some OSDs are handling disproportionate client load.

## Use rados list to Find Heavy Objects

Identify pools with the most active objects:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash -c "
  rados df
"
```

This shows object count and read/write operations per pool:

```
POOL_NAME         USED  OBJECTS  CLONES  COPIES  MISSING_ON_PRIMARY
rbd              50GiB    12500       0   37500                   0
cephfs-data      20GiB     4200       0   12600                   0
```

## Correlate Kubernetes Workloads with Ceph Pools

Map PVCs to RBD images to find which workloads are driving IO:

```bash
# List all RBD images in a pool
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rbd ls --pool rbd

# Get image details including size and parent
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rbd info rbd/<image-name>

# Check PVC to image mapping
kubectl get pv -o json | jq -r '.items[] | [.metadata.name, .spec.csi.volumeAttributes.imageName] | @tsv'
```

## Summary

Tracking client IO distribution in Ceph combines per-pool statistics from `ceph osd pool stats`, OSD-level `ceph osd perf` metrics, and Prometheus CSI metrics to identify which workloads, pools, and OSDs carry the heaviest IO load. Correlating Kubernetes PVCs with RBD images lets you pinpoint specific applications driving high utilization and enables informed decisions around QoS, pool placement, and capacity expansion.
