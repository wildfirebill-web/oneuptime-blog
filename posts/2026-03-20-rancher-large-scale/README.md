# How to Configure Rancher for Large-Scale Deployments

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Large Scale, Enterprise, Performance

Description: Configure Rancher to manage large-scale Kubernetes deployments with hundreds of clusters and thousands of nodes through architecture, resource planning, and operational practices.

## Introduction

Rancher is capable of managing hundreds of Kubernetes clusters from a single control plane. However, achieving this scale requires careful architecture decisions, proper resource allocation, and operational discipline. This guide covers the key considerations for deploying Rancher at scale.

## Prerequisites

- Rancher v2.7+ (latest stable recommended)
- Dedicated infrastructure for Rancher's local cluster
- External monitoring and alerting
- Network connectivity between Rancher and all managed clusters

## Step 1: Architecture for Scale

```text
Large-Scale Rancher Architecture:
├── Rancher Local Cluster (RKE2, 3-5 control plane nodes)
│   ├── etcd nodes: 3 dedicated m5.2xlarge nodes (NVMe SSD)
│   ├── Control plane: 3 dedicated nodes
│   └── Rancher pods: 3-5 replicas
│
├── Region A Clusters (50+ clusters)
│   ├── Production clusters
│   ├── Staging clusters
│   └── Development clusters
│
├── Region B Clusters (50+ clusters)
│   └── ... same pattern
│
└── Fleet GitOps (manages all clusters)
```

```bash
# Rancher supports up to 2000 clusters in a single instance

# (with proper hardware and configuration)

# Recommended hardware for 100+ cluster deployments:
# - Rancher server: 8 CPU, 16GB RAM per replica
# - etcd nodes: 4 CPU, 8GB RAM, NVMe SSD

# Check current cluster count
kubectl get clusters.management.cattle.io --all-namespaces | wc -l
```

## Step 2: Configure Rancher Server for Scale

```yaml
# rancher-scale-values.yaml - Helm values for large deployments
replicas: 5

resources:
  requests:
    cpu: 4000m
    memory: 8Gi
  limits:
    cpu: 8000m
    memory: 16Gi

extraEnv:
  # Increase workers for parallel operations
  - name: CATTLE_WORKERS
    value: "50"

  # Increase connection pool
  - name: CATTLE_DB_CATTLE_MAX_POOL_SIZE
    value: "200"

  # Reduce reconciliation frequency
  - name: CATTLE_RESYNC_DEFAULT
    value: "1800"  # 30 minutes

  # Disable unnecessary features
  - name: CATTLE_FEATURES
    value: "fleet=true,continuous-delivery=true,uiextension=true"

  # Increase Golang runtime
  - name: GOMAXPROCS
    value: "16"
  - name: GOGC
    value: "100"

# Node affinity for dedicated Rancher nodes
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
            - key: app
              operator: In
              values:
                - rancher
        topologyKey: kubernetes.io/hostname
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: rancher-role
              operator: In
              values:
                - rancher-server
```

## Step 3: Scale the Local Cluster etcd

```yaml
# rke2-config-scale.yaml - etcd tuning for 100+ managed clusters
# /etc/rancher/rke2/config.yaml on control plane nodes

etcd-arg:
  - "quota-backend-bytes=17179869184"  # 16GB quota
  - "auto-compaction-mode=periodic"
  - "auto-compaction-retention=1h"
  - "max-request-bytes=33554432"  # 32MB
  - "snapshot-count=5000"

# Separate etcd nodes from control plane
# Use node labels and taints
```

```bash
# Taint etcd nodes for dedicated use
kubectl taint nodes etcd-node-01 etcd-only=true:NoSchedule
kubectl label nodes etcd-node-01 node-role.kubernetes.io/etcd=true

# Taint control plane nodes
kubectl taint nodes cp-node-01 control-plane=true:NoSchedule
kubectl label nodes cp-node-01 node-role.kubernetes.io/master=true
```

## Step 4: Configure Fleet for Scale

```yaml
# gitrepo-scale.yaml - Fleet configuration for many clusters
apiVersion: fleet.cattle.io/v1alpha1
kind: GitRepo
metadata:
  name: production-apps
  namespace: fleet-default
spec:
  repo: https://github.com/company/k8s-configs
  branch: main
  # Limit parallel updates to avoid overwhelming API server
  concurrency: 10
  targets:
    - name: all-production
      clusterSelector:
        matchLabels:
          env: production
```

## Step 5: Implement Cluster Grouping

```bash
# Use Fleet ClusterGroups for organized management
cat <<EOF | kubectl apply -f -
apiVersion: fleet.cattle.io/v1alpha1
kind: ClusterGroup
metadata:
  name: us-east-production
  namespace: fleet-default
spec:
  selector:
    matchLabels:
      env: production
      region: us-east
EOF

# Apply different configs to different groups
# Use GitRepo targets to filter by cluster labels
```

## Step 6: Implement Observability at Scale

```yaml
# rancher-monitoring-scale.yaml - Monitoring tuned for scale
prometheus:
  prometheusSpec:
    retention: 7d  # Reduce retention for scale
    resources:
      requests:
        cpu: 2000m
        memory: 8Gi
    # Shard Prometheus for scale
    shards: 3
    # Remote write to Thanos for long-term storage
    remoteWrite:
      - url: http://thanos-receive:10908/api/v1/receive

grafana:
  resources:
    requests:
      memory: 1Gi
    limits:
      memory: 2Gi
```

## Step 7: Network Considerations

```bash
# Each cluster agent maintains a websocket connection to Rancher
# Ensure your load balancer supports 1000+ concurrent websockets
# Configure connection timeouts appropriately

# Check active connections to Rancher
kubectl exec -n cattle-system deployment/rancher -- \
  netstat -an | grep ESTABLISHED | wc -l

# Each cluster uses approximately 2-5 MB/s for normal operations
# Plan network capacity accordingly
```

## Conclusion

Running Rancher at large scale requires dedicated infrastructure, carefully tuned configuration, and solid operational practices. The most critical factors are properly sized etcd storage on fast SSDs, adequate CPU and memory for Rancher server replicas, and network infrastructure that supports many simultaneous websocket connections. Use Fleet for GitOps-based cluster management at scale, and implement comprehensive monitoring to detect performance degradation early.
