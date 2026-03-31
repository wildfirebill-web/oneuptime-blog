# How to Configure Ceph for Hyper-Converged Infrastructure

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, HCI, Hyper-Converged, Kubernetes, Infrastructure

Description: Configure Rook-Ceph on hyper-converged infrastructure where storage and compute share the same nodes, with proper resource isolation to avoid contention.

---

## What is HCI with Rook-Ceph

Hyper-converged infrastructure (HCI) runs storage daemons (OSDs, MONs) on the same nodes as application workloads. This reduces hardware costs but requires careful resource management to prevent storage I/O from starving compute workloads and vice versa.

## Resource Reservation for Storage Daemons

```yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  resources:
    osd:
      requests:
        cpu: "1"
        memory: "3Gi"
      limits:
        cpu: "2"
        memory: "6Gi"
    mon:
      requests:
        cpu: "250m"
        memory: "512Mi"
      limits:
        cpu: "1"
        memory: "1Gi"
    mgr:
      requests:
        cpu: "250m"
        memory: "256Mi"
      limits:
        cpu: "1"
        memory: "1Gi"
```

## Reserve Node Resources for Ceph

Use Kubernetes node allocatable reservations to protect system and storage resources:

```yaml
# Add to kubelet configuration on storage nodes
# /var/lib/kubelet/config.yaml
evictionHard:
  memory.available: "1Gi"
  nodefs.available: "10%"
systemReserved:
  cpu: "500m"
  memory: "2Gi"
kubeReserved:
  cpu: "500m"
  memory: "1Gi"
```

## Topology Spread for Application Pods

Ensure application pods spread across all HCI nodes for even storage load:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: myapp
```

## Anti-Affinity for Critical Storage Daemons

Keep OSDs and application pods from competing on the same node:

```yaml
# In CephCluster spec
spec:
  placement:
    osd:
      podAntiAffinity:
        preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: high-cpu-workload
              topologyKey: kubernetes.io/hostname
```

## Configure cgroups for I/O Isolation

```bash
# Set I/O weight for Ceph processes (higher = more I/O priority)
# Ceph OSD needs higher I/O priority than background apps
echo 500 > /sys/fs/cgroup/blkio/kubepods/besteffort/blkio.weight

# OSD pods should be guaranteed QoS (not burstable or BestEffort)
# This happens automatically when requests == limits
```

## Monitor Resource Contention

```bash
# Check if OSD latency spikes correlate with app CPU usage
# High OSD apply latency during app workload = resource contention

ceph osd perf | sort -k3 -n | tail -5

# Monitor node-level resource saturation
kubectl top nodes
kubectl top pods -n rook-ceph

# Check for OSD evictions
kubectl get events -n rook-ceph | grep Evicted
```

## Summary

HCI with Rook-Ceph requires explicit CPU and memory requests/limits on all storage daemons, node-level resource reservations to prevent OOM evictions, and topology constraints that spread application load evenly. Proper resource isolation ensures storage performance remains predictable even during heavy application workloads.
