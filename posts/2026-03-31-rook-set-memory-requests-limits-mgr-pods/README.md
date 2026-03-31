# How to Set Memory Requests and Limits for Rook-Ceph MGR Pods

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Kubernetes, Manager, Memory, Resource, Pod

Description: Configure memory requests and limits for Rook-Ceph MGR (manager) pods in Kubernetes to support dashboard, Prometheus, and orchestration modules without OOM kills.

---

## Overview

The Ceph Manager (MGR) daemon runs multiple modules including the dashboard, Prometheus exporter, balancer, and orchestrator. Each module consumes memory, and the MGR can grow significantly in large clusters. Proper memory configuration prevents OOM kills that would disrupt monitoring and cluster management.

## MGR Memory Consumers

Key memory consumers in the MGR pod:
- **Dashboard module** - 200-500 MB depending on cluster size
- **Prometheus module** - 50-200 MB, grows with cluster size
- **Balancer module** - minimal
- **Orchestrator/Rook module** - 100-300 MB
- **Ceph MGR base process** - 200-500 MB
- **Total typical range** - 512 MB to 2 GB

## Configuring MGR Memory Resources

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
        cpu: "1000m"
        memory: "1Gi"
    mgr-sidecar:
      requests:
        cpu: "100m"
        memory: "40Mi"
      limits:
        cpu: "200m"
        memory: "100Mi"
```

Apply and verify:

```bash
kubectl apply -f cephcluster.yaml

# Check MGR pod status after restart
kubectl -n rook-ceph get pods -l app=rook-ceph-mgr

# Verify resource limits
kubectl -n rook-ceph describe pod rook-ceph-mgr-<node>-<hash> | grep -A10 "Limits:"
```

## Monitoring MGR Memory Usage

```bash
# Real-time memory usage
kubectl -n rook-ceph top pods -l app=rook-ceph-mgr

# Check for OOM kills
kubectl -n rook-ceph describe pod rook-ceph-mgr-<node>-<hash> | grep -i "oomkill\|OOMKilled"

# Check OOM history via events
kubectl -n rook-ceph get events | grep -i "oom\|OOMKilled" | grep mgr
```

## Prometheus Query for MGR Memory

```promql
# MGR memory usage over time
container_memory_working_set_bytes{
  namespace="rook-ceph",
  pod=~"rook-ceph-mgr-.*"
}

# MGR memory limit percentage
container_memory_working_set_bytes{
  namespace="rook-ceph",
  pod=~"rook-ceph-mgr-.*"
} /
container_spec_memory_limit_bytes{
  namespace="rook-ceph",
  pod=~"rook-ceph-mgr-.*"
}
```

## Memory Sizing by Cluster Size

| Cluster Size | MGR Memory Request | MGR Memory Limit |
|---|---|---|
| Small (< 10 OSDs) | 512Mi | 1Gi |
| Medium (10-50 OSDs) | 1Gi | 2Gi |
| Large (50-200 OSDs) | 1Gi | 3Gi |
| Extra Large (> 200 OSDs) | 2Gi | 4Gi |

## Reducing MGR Memory Footprint

```bash
# Disable unused modules to reduce memory
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
    ceph mgr module disable insights

kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
    ceph mgr module disable crash

# Check which modules are enabled
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
    ceph mgr module ls
```

## MGR High Availability

Rook runs a standby MGR for high availability. Configure resources for both active and standby:

```bash
# Both active and standby use the same resource spec
# Verify both are running
kubectl -n rook-ceph get pods -l app=rook-ceph-mgr

# Check which is active
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph mgr stat
```

## Summary

MGR pod memory requirements grow with cluster size and enabled modules. Start with 512Mi request and 1Gi limit for small clusters, scaling up to 2Gi/4Gi for large deployments. Disable unused MGR modules to reduce baseline memory consumption. Monitor memory usage via Prometheus and increase limits before OOM kills occur, as an OOM-killed MGR disrupts dashboards, alerting, and the balancer.
