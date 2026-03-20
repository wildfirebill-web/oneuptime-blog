# How to Design Rancher Architecture for Production

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Architecture, Production, High Availability, Kubernetes, Design

Description: Design a production-grade Rancher architecture with high availability, multi-cluster management, network topology, storage strategy, and security considerations for enterprise deployments.

## Introduction

A production Rancher architecture must address high availability, scalability, security, and operational simplicity. Poor architectural decisions—single-point-of-failure management planes, undersized etcd clusters, flat network topologies—lead to outages and security incidents. This guide covers the key architectural decisions for a production Rancher deployment.

## Reference Architecture

```
                    ┌─────────────────────────────┐
                    │    Global Load Balancer      │
                    │    (AWS ALB / F5 / NGINX)    │
                    └──────────────┬──────────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              │                    │                    │
    ┌─────────▼──────┐   ┌─────────▼──────┐   ┌────────▼───────┐
    │  Rancher Node 1│   │  Rancher Node 2│   │  Rancher Node 3│
    │  (RKE2 CP)     │   │  (RKE2 CP)     │   │  (RKE2 CP)     │
    └────────────────┘   └────────────────┘   └────────────────┘
              │                    │                    │
              └────────────────────┼────────────────────┘
                                   │ manages
              ┌────────────────────┼────────────────────┐
              │                    │                    │
    ┌─────────▼──────┐   ┌─────────▼──────┐   ┌────────▼───────┐
    │  Production    │   │  Staging       │   │  Dev/Test      │
    │  Cluster       │   │  Cluster       │   │  Clusters      │
    └────────────────┘   └────────────────┘   └────────────────┘
```

## Decision 1: Rancher Management Cluster

Use a dedicated RKE2 cluster (not a downstream workload cluster) for Rancher:

```yaml
# rke2-rancher-management.yaml
nodes:
  - address: rancher-1.internal.com
    user: rke2
    roles: [controlplane, etcd]
  - address: rancher-2.internal.com
    user: rke2
    roles: [controlplane, etcd]
  - address: rancher-3.internal.com
    user: rke2
    roles: [controlplane, etcd]

# Sizing guidelines for management cluster:
# Up to 150 clusters / 1500 nodes: 4 vCPU, 16 GB RAM per node
# Up to 300 clusters / 3000 nodes: 8 vCPU, 32 GB RAM per node
# Up to 500 clusters / 5000 nodes: 16 vCPU, 64 GB RAM per node
```

## Decision 2: etcd Configuration

```bash
# Dedicated etcd nodes for high cluster counts
# etcd should have dedicated SSDs for low latency

# Recommended etcd disk performance
# - 99th percentile commit latency < 10ms
# - Use NVMe SSDs in production

# Verify etcd latency
etcdctl check perf --endpoints=https://etcd-1:2379
```

## Decision 3: Network Architecture

```yaml
# Separate networks for security and performance
networks:
  management:     10.0.0.0/24    # Rancher management plane
  pod:            10.244.0.0/16  # Pod network (per cluster)
  service:        10.96.0.0/12   # Service CIDR (per cluster)
  storage:        10.0.1.0/24    # Storage replication traffic

# CNI selection:
# - Calico: BGP mode for high performance, no encapsulation overhead
# - Flannel: Simpler, VXLAN mode for cross-subnet
# - Cilium: eBPF-based, best for observability and security policies
```

## Decision 4: Storage Strategy

```yaml
# Longhorn for Rancher-native distributed storage
helm install longhorn longhorn/longhorn \
  --namespace longhorn-system \
  --set defaultSettings.defaultReplicaCount=3 \
  --set defaultSettings.storageMinimalAvailablePercentage=15 \
  --set defaultSettings.storageReservedPercentageForDefaultDisk=25

# For databases: dedicated storage class with local SSDs
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-ssd
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```

## Decision 5: Multi-Tenancy Model

```yaml
# Rancher Project = namespace grouping + RBAC boundary
# Each team gets a Project with ResourceQuota

apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
  namespace: team-a-production
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 40Gi
    limits.cpu: "40"
    limits.memory: 80Gi
    persistentvolumeclaims: "20"
    services.loadbalancers: "5"
```

## Decision 6: Backup and DR

```bash
# Rancher backup (management plane)
helm install rancher-backup rancher-charts/rancher-backup \
  --namespace cattle-resources-system \
  --set persistence.enabled=true \
  --set persistence.storageClass=longhorn

# Create recurring backup
kubectl apply -f - <<EOF
apiVersion: resources.cattle.io/v1
kind: Backup
metadata:
  name: rancher-daily-backup
spec:
  storageLocation:
    s3:
      bucketName: rancher-backups
      region: us-east-1
  schedule: "0 2 * * *"       # Daily at 2 AM
  retentionCount: 14           # Keep 14 backups
EOF
```

## Production Checklist

- 3-node Rancher management cluster (HA)
- Dedicated etcd on NVMe SSDs
- External load balancer with health checks
- Separate management, pod, and storage networks
- Longhorn with 3 replicas for stateful workloads
- Daily automated backups to S3
- Monitoring stack (Prometheus + Grafana) deployed
- RBAC roles aligned to team structure
- Network policies enforcing namespace isolation
- cert-manager managing TLS certificates

## Conclusion

Production Rancher architecture requires careful planning across HA, networking, storage, and security dimensions. A 3-node management cluster running on RKE2 with dedicated etcd SSDs handles hundreds of downstream clusters reliably. Combining Longhorn storage, Fleet GitOps, and Prometheus monitoring creates a self-contained, operationally simple platform for enterprise Kubernetes management.
