# How to Create CAPI Clusters with Rancher Turtles

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher Turtles, CAPI, Kubernetes, Rancher, Cluster Provisioning

Description: Create and manage Kubernetes clusters using Cluster API resources through Rancher Turtles with declarative YAML manifests.

## Introduction

With Rancher Turtles installed, you can create Kubernetes clusters by applying standard Cluster API (CAPI) resources. These clusters are automatically imported into Rancher, giving you centralized management across all your CAPI-provisioned clusters.

## Prerequisites

- Rancher Turtles installed
- At least one CAPI infrastructure provider installed
- Bootstrap and control plane providers configured
- Infrastructure credentials configured

## Cluster API Resource Structure

A CAPI cluster requires these resources:

```text
Cluster (cluster.x-k8s.io)
├── InfrastructureCluster (e.g., AWSCluster)
├── ControlPlane (e.g., RKE2ControlPlane)
└── MachineDeployments
    ├── MachineTemplate (e.g., AWSMachineTemplate)
    └── BootstrapConfigTemplate (e.g., RKE2ConfigTemplate)
```

## Creating a Cluster with Docker Provider (Testing)

The Docker provider is useful for testing CAPI workflows:

```bash
# Initialize Docker provider

clusterctl init --infrastructure docker

# Generate cluster manifest
clusterctl generate cluster my-test-cluster \
  --infrastructure docker \
  --kubernetes-version v1.28.0 \
  --control-plane-machine-count 1 \
  --worker-machine-count 2 \
  > my-test-cluster.yaml

# Apply the cluster
kubectl apply -f my-test-cluster.yaml
```

## Creating a Production Cluster

```yaml
# production-cluster.yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  name: production-cluster
  namespace: default
  labels:
    # Enable auto-import into Rancher
    cluster-api.cattle.io/rancher-auto-import: "true"
spec:
  clusterNetwork:
    pods:
      cidrBlocks:
        - 10.244.0.0/16
    services:
      cidrBlocks:
        - 10.96.0.0/12
  infrastructureRef:
    apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
    kind: DockerCluster
    name: production-cluster
  controlPlaneRef:
    apiVersion: controlplane.cluster.x-k8s.io/v1alpha1
    kind: RKE2ControlPlane
    name: production-cluster-cp
---
apiVersion: controlplane.cluster.x-k8s.io/v1alpha1
kind: RKE2ControlPlane
metadata:
  name: production-cluster-cp
  namespace: default
spec:
  replicas: 3
  version: v1.28.0+rke2r1
  nodeDrainTimeout: 5m
  rolloutStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
```

## Monitoring Cluster Provisioning

```bash
# Watch cluster status
kubectl get cluster production-cluster --watch

# Check all CAPI resources
kubectl get clusters,machines,machinedeployments -n default

# View control plane status
kubectl get rke2controlplane -n default

# Get cluster kubeconfig once ready
clusterctl get kubeconfig production-cluster > production-kubeconfig.yaml

# Test connectivity to new cluster
export KUBECONFIG=production-kubeconfig.yaml
kubectl get nodes
```

## Verifying Import into Rancher

```bash
# Check if Rancher has imported the cluster
kubectl get cluster.provisioning.cattle.io \
  -n fleet-default | grep production-cluster

# Or check in Rancher UI:
# Cluster Management > Clusters > production-cluster
```

## Scaling the Cluster

```bash
# Scale worker nodes
kubectl scale machinedeployment production-cluster-workers \
  --replicas=5

# Or patch the MachineDeployment
kubectl patch machinedeployment production-cluster-workers \
  --type merge \
  -p '{"spec":{"replicas":5}}'
```

## Conclusion

Creating clusters with Rancher Turtles combines the declarative power of Cluster API with Rancher's management capabilities. CAPI's resource model provides fine-grained control over every aspect of cluster infrastructure, while Rancher Turtles' auto-import feature ensures all clusters are visible and manageable through the Rancher UI.
