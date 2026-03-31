# How to Set Resource Requests and Limits for Rook-Ceph Components

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Resource Management, Kubernetes, Performance

Description: Learn how to configure CPU and memory resource requests and limits for Rook-Ceph components (MON, MGR, OSD, MDS, RGW) in the CephCluster CRD.

---

## Why Resource Management Matters for Ceph

Without resource requests and limits, Ceph daemon pods can:
- Starve other workloads by consuming all node CPU
- Be OOM-killed by Kubernetes when memory pressure occurs
- Not be scheduled properly due to missing resource hints

Proper resource configuration ensures Ceph daemons are scheduled predictably, protected from eviction, and do not monopolize node resources.

## Resource Configuration Structure

Resources are configured in the `CephCluster` spec under the `resources` key, organized by daemon type:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  resources:
    mgr:
      requests:
        cpu: "500m"
        memory: "512Mi"
      limits:
        cpu: "1"
        memory: "1Gi"
    mon:
      requests:
        cpu: "500m"
        memory: "1Gi"
      limits:
        cpu: "1"
        memory: "2Gi"
    osd:
      requests:
        cpu: "1"
        memory: "4Gi"
      limits:
        cpu: "2"
        memory: "8Gi"
    mds:
      requests:
        cpu: "500m"
        memory: "512Mi"
      limits:
        cpu: "2"
        memory: "4Gi"
    rgw:
      requests:
        cpu: "500m"
        memory: "512Mi"
      limits:
        cpu: "2"
        memory: "2Gi"
    mgr-sidecar:
      requests:
        cpu: "100m"
        memory: "40Mi"
      limits:
        cpu: "500m"
        memory: "100Mi"
```

## Per-Component Guidelines

### MON (Monitor) Resources

Monitors run a LevelDB/RocksDB store for cluster state. Memory requirements depend on PG count:

```yaml
mon:
  requests:
    cpu: "250m"     # Low CPU at idle, bursts during elections
    memory: "1Gi"   # For clusters with <1000 PGs
  limits:
    cpu: "1"
    memory: "2Gi"   # For clusters with <10000 PGs
```

For large clusters (10000+ PGs):

```yaml
mon:
  limits:
    memory: "4Gi"
```

### OSD Resources

OSDs are the most resource-intensive component. Each OSD process needs memory for its RocksDB cache:

```yaml
osd:
  requests:
    cpu: "500m"
    memory: "2Gi"   # Minimum for a small OSD
  limits:
    cpu: "2"
    memory: "8Gi"   # For a 1-4Ti OSD
```

The OSD memory target is configurable in Ceph:

```bash
# Set OSD memory target (Ceph-level, not resource limit)
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set osd osd_memory_target 4294967296  # 4Gi per OSD
```

The Kubernetes memory limit should be higher than `osd_memory_target` (add 25-50% buffer).

### MGR Resources

```yaml
mgr:
  requests:
    cpu: "500m"
    memory: "512Mi"
  limits:
    cpu: "1"
    memory: "1Gi"
```

If using the Prometheus module or many dashboard users, increase memory limits.

### MDS (CephFS Metadata Server) Resources

MDS caches filesystem metadata. More cache = better CephFS performance:

```yaml
mds:
  requests:
    cpu: "500m"
    memory: "1Gi"
  limits:
    cpu: "2"
    memory: "4Gi"   # Larger clusters benefit from more cache
```

### RGW (Object Storage Gateway) Resources

```yaml
rgw:
  requests:
    cpu: "500m"
    memory: "512Mi"
  limits:
    cpu: "2"
    memory: "2Gi"   # Scale up for high request rates
```

## Development vs Production Presets

### Development/Testing (Minimal Resources)

```yaml
resources:
  mgr:
    requests:
      cpu: "125m"
      memory: "256Mi"
    limits:
      cpu: "500m"
      memory: "512Mi"
  mon:
    requests:
      cpu: "125m"
      memory: "256Mi"
    limits:
      cpu: "500m"
      memory: "512Mi"
  osd:
    requests:
      cpu: "250m"
      memory: "512Mi"
    limits:
      cpu: "500m"
      memory: "2Gi"
```

### Production (Recommended)

```yaml
resources:
  mgr:
    requests:
      cpu: "500m"
      memory: "512Mi"
    limits:
      cpu: "2"
      memory: "2Gi"
  mon:
    requests:
      cpu: "500m"
      memory: "1Gi"
    limits:
      cpu: "2"
      memory: "4Gi"
  osd:
    requests:
      cpu: "2"
      memory: "4Gi"
    limits:
      cpu: "4"
      memory: "12Gi"
```

## Monitoring Actual Resource Usage

```bash
# Check current resource usage of Ceph pods
kubectl -n rook-ceph top pods | sort -k3 -hr | head -20

# Check for OOM kills
kubectl -n rook-ceph get events | grep -i oomkill

# Check resource requests vs limits in detail
kubectl -n rook-ceph describe pod <osd-pod> | grep -A4 "Limits\|Requests"
```

## Updating Resources on a Running Cluster

Resource changes trigger pod restarts for affected daemons:

```bash
# Update the CephCluster resource configuration
kubectl -n rook-ceph edit cephcluster rook-ceph
# Change resources section and save

# Watch for rolling restarts
kubectl -n rook-ceph get pods -w
```

## Summary

Resource requests and limits for Rook-Ceph components are configured under `spec.resources` in the `CephCluster` CRD, organized by daemon type (mon, mgr, osd, mds, rgw). Set requests to ensure proper scheduling and limits to protect the node from memory exhaustion. OSD memory limits should be at least 25% higher than the Ceph `osd_memory_target` setting. Monitor actual resource usage with `kubectl top pods` and adjust based on observed consumption rather than fixed formulas.
