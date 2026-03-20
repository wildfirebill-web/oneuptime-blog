# How to Configure Rancher for Large-Scale Deployments - Deployments

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Large Scale, Enterprise, Performance, Kubernetes, Architecture

Description: Configure Rancher for large-scale deployments with hundreds of clusters by tuning server resources, optimizing etcd, enabling external databases, and implementing proper HA.

## Introduction

Running Rancher at large scale-100+ downstream clusters, thousands of nodes-requires architectural decisions beyond the default installation. This guide covers the critical configurations needed to maintain Rancher Server stability and performance at enterprise scale.

## Rancher Scalability Guidelines

| Deployment Scale | Clusters | Nodes | Rancher Server Resources |
|---|---|---|---|
| Small | 1-50 | 1-500 | 4 CPU, 8GB RAM |
| Medium | 50-200 | 500-2000 | 8 CPU, 16GB RAM |
| Large | 200-1000 | 2000-10000 | 16 CPU, 32GB RAM |
| Enterprise | 1000+ | 10000+ | 32 CPU, 64GB RAM + External DB |

## Step 1: External Database for Large Scale

At 200+ clusters, migrate from the embedded SQLite to PostgreSQL:

```yaml
# rancher-values.yaml

extraEnv:
  - name: CATTLE_DB_CATTLE_MYSQL_HOST
    value: "postgres.databases.svc.cluster.local"
  - name: CATTLE_DB_CATTLE_MYSQL_USER
    valueFrom:
      secretKeyRef:
        name: rancher-db-credentials
        key: username
  - name: CATTLE_DB_CATTLE_MYSQL_PASSWORD
    valueFrom:
      secretKeyRef:
        name: rancher-db-credentials
        key: password
  - name: CATTLE_DB_CATTLE_MYSQL_NAME
    value: "rancher"
```

## Step 2: Tune Rancher Reconciliation Loops

Reduce CPU overhead from frequent reconciliation:

```yaml
extraEnv:
  # How often Rancher re-syncs all cluster state
  - name: CATTLE_RESYNC_DEFAULT
    value: "60"    # Increase from default 15s to 60s
  - name: CATTLE_CLUSTER_AGENT_RESYNC
    value: "300"   # Agent resync every 5 minutes
  # Limit concurrent processing
  - name: CATTLE_WORKER_COUNT
    value: "100"   # Adjust based on CPU capacity
```

## Step 3: Scale etcd for Large Object Counts

Large clusters with many Custom Resources can exceed etcd defaults:

```yaml
# RKE2 etcd configuration
etcd-arg:
  - "quota-backend-bytes=12884901888"    # 12GB quota for large deployments
  - "auto-compaction-retention=4"         # Compact every 4 hours
  - "max-request-bytes=10485760"          # 10MB max request size
  - "heartbeat-interval=500"
  - "election-timeout=5000"
```

## Step 4: Distribute Load with Multiple Rancher Replicas

```yaml
# rancher-values.yaml
replicas: 5    # 5 Rancher server pods

# Configure horizontal pod autoscaling
autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 60
```

## Step 5: Separate Cluster-Level etcd

For the local Rancher cluster, use dedicated etcd nodes:

```yaml
# rke2-config.yaml
nodes:
  # Dedicated etcd-only nodes (no workloads)
  - role: [etcd]
    address: 10.0.0.10
    taints:
      - effect: NoSchedule
        key: node-role.kubernetes.io/etcd
  # Control plane nodes (no etcd, no workloads)
  - role: [controlplane]
    address: 10.0.0.20
```

## Step 6: Implement Cluster Import Rate Limiting

When importing many clusters simultaneously, stagger the imports:

```bash
# Import clusters in batches with delays
for cluster in cluster-{1..100}; do
  rancher cluster import $cluster
  sleep 10    # Allow Rancher to process each import
done
```

## Conclusion

Large-scale Rancher deployments require an external database, increased server resources, tuned reconciliation frequencies, and dedicated infrastructure for the local cluster's etcd. At 1000+ cluster scale, work with SUSE Rancher Support to review your specific architecture and receive guidance on Rancher Prime features designed for enterprise scale.
