# How to Plan Cluster Topology in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Cluster Topology, Architecture, Kubernetes, Multi-Cluster, Planning

Description: Plan cluster topology in Rancher to balance workload isolation, resource efficiency, compliance requirements, and operational complexity for multi-team and multi-environment Kubernetes deployments.

## Introduction

Cluster topology-how many clusters to run, what workloads to put in each, and how to organize namespaces-is one of the most consequential architectural decisions in a Rancher deployment. Too few clusters create blast radius risks; too many create operational complexity. The right topology balances isolation requirements with management overhead.

## Topology Models

### Model 1: Environment-Based (Recommended Starting Point)

```text
Rancher Management Cluster
├── Production Cluster
│   ├── namespace: team-a
│   ├── namespace: team-b
│   └── namespace: team-c
├── Staging Cluster
│   └── (mirrors production namespaces)
└── Development Cluster
    └── (developer self-service)
```

### Model 2: Team-Based (Better Isolation)

```text
Rancher Management Cluster
├── Team A Cluster (production + staging)
├── Team B Cluster (production + staging)
├── Shared Services Cluster (logging, monitoring, CI/CD)
└── Development Cluster (shared)
```

### Model 3: Compliance-Driven

```text
Rancher Management Cluster
├── PCI Cluster (payment processing - isolated)
├── HIPAA Cluster (healthcare data - isolated)
├── General Production Cluster
└── Non-Production Cluster
```

## Step 1: Assess Isolation Requirements

```bash
# Questions to guide topology decisions:

# 1. Compliance: Do any workloads require dedicated infrastructure?

#    - PCI-DSS: Requires network isolation for cardholder data
#    - HIPAA: PHI must be isolated from non-PHI workloads
#    - FedRAMP: Government workloads may require air-gapped clusters

# 2. Team boundaries: Do teams need cluster-level RBAC separation?
#    - Namespace RBAC in Rancher Projects handles most multi-team needs
#    - Separate clusters only when teams have conflicting requirements

# 3. Blast radius: What's the acceptable failure domain?
#    - One cluster failure should never affect all environments
#    - Production should never share a cluster with development
```

## Step 2: Size Your Clusters

```yaml
# Cluster sizing guidelines

# Small cluster (dev/test):
#   3 control plane nodes: 4 vCPU, 8 GB RAM
#   3-10 worker nodes: 4-8 vCPU, 16-32 GB RAM
#   Use case: development, CI/CD, testing

# Medium cluster (staging):
#   3 control plane nodes: 4 vCPU, 16 GB RAM
#   5-20 worker nodes: 8-16 vCPU, 32-64 GB RAM
#   Use case: staging, UAT, load testing

# Large cluster (production):
#   3-5 control plane nodes: 8 vCPU, 32 GB RAM
#   20-100 worker nodes: 16-32 vCPU, 64-128 GB RAM
#   Use case: production workloads, scale-tested

# Rancher management cluster:
#   3 nodes: 8 vCPU, 32 GB RAM (for up to 300 downstream clusters)
```

## Step 3: Plan Node Pools

```yaml
# RKE2 cluster with specialized node pools
apiVersion: provisioning.cattle.io/v1
kind: Cluster
metadata:
  name: production
  namespace: fleet-default
spec:
  rkeConfig:
    nodePools:
      # Control plane pool
      - name: control-plane
        quantity: 3
        machineConfigRef:
          kind: VmwarevsphereConfig
          name: control-plane-vm
        roles: [controlplane, etcd]

      # General workload pool
      - name: workers-general
        quantity: 5
        machineConfigRef:
          kind: VmwarevsphereConfig
          name: worker-vm
        roles: [worker]

      # Memory-optimized pool for databases
      - name: workers-db
        quantity: 3
        machineConfigRef:
          kind: VmwarevsphereConfig
          name: worker-memory-vm
        roles: [worker]
        labels:
          workload-type: database
```

## Step 4: Namespace Organization

```yaml
# Rancher Projects organize namespaces with shared RBAC and quotas
# Recommended namespace naming convention:
#   {team}-{environment}  e.g., payments-prod, payments-staging

# Create a Project for each team in Rancher
# Project maps to ResourceQuota + LimitRange + RBAC roles

apiVersion: management.cattle.io/v3
kind: Project
metadata:
  name: payments-team
spec:
  displayName: "Payments Team"
  namespaces:
    - payments-prod
    - payments-staging
  resourceQuota:
    limit:
      limitsCpu: "40"
      limitsMemory: "80Gi"
```

## Step 5: Plan for Cross-Cluster Communication

```yaml
# For services that need to communicate across clusters:
# Option 1: Expose via LoadBalancer + external DNS
# Option 2: Service mesh (Istio multicluster)
# Option 3: Submariner for cross-cluster pod networking

# Shared services (monitoring, logging) pattern:
# Deploy central Prometheus in management cluster
# Use remote_write from downstream clusters
prometheus:
  remoteWrite:
    - url: "https://prometheus.management.company.com/api/v1/write"
      basicAuth:
        username:
          name: prometheus-remote-write-secret
          key: username
        password:
          name: prometheus-remote-write-secret
          key: password
```

## Conclusion

Cluster topology in Rancher should start simple (environment-based) and evolve toward team-based or compliance-driven models only when isolation requirements demand it. Use Rancher Projects and namespace-level RBAC to provide team isolation within shared clusters, reserving separate clusters for hard compliance boundaries. Size clusters based on growth projections, and always maintain a dedicated Rancher management cluster separate from workload clusters.
