# How to Troubleshoot Rancher Turtles Cluster Provisioning

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher Turtles, CAPI, Troubleshooting, Kubernetes, Debugging

Description: Debug and resolve common cluster provisioning failures when using Rancher Turtles and Cluster API.

## Introduction

How to Troubleshoot Rancher Turtles Cluster Provisioning is an important aspect of managing Kubernetes clusters with Rancher Turtles and Cluster API. This guide provides a comprehensive walkthrough with practical examples and best practices.

## Prerequisites

- Rancher Turtles installed and configured
- kubectl access to the management cluster
- Appropriate cloud provider credentials (if applicable)
- Cluster API providers installed

## Overview

Rancher Turtles integrates Cluster API (CAPI) with Rancher to provide a unified, declarative approach to Kubernetes cluster lifecycle management. This guide walks through the specifics of How to Troubleshoot Rancher Turtles Cluster Provisioning.

## Step 1: Prepare Your Environment

```bash
# Verify Rancher Turtles is running

kubectl get pods -n rancher-turtles-system

# Check installed CAPI providers
kubectl get providers -A

# Verify management cluster connectivity
kubectl cluster-info
```

## Step 2: Configure Resources

```yaml
# Example CAPI configuration for How to Troubleshoot Rancher Turtles Cluster Provisioning
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  name: example-cluster
  namespace: default
  labels:
    cluster-api.cattle.io/rancher-auto-import: "true"
    environment: production
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
    kind: InfraCluster
    name: example-cluster
  controlPlaneRef:
    apiVersion: controlplane.cluster.x-k8s.io/v1alpha1
    kind: RKE2ControlPlane
    name: example-cluster-cp
```

```bash
# Apply the configuration
kubectl apply -f cluster-config.yaml

# Monitor progress
kubectl get cluster example-cluster --watch
```

## Step 3: Verify the Configuration

```bash
# Check cluster status
kubectl get clusters -A

# Describe the cluster for detailed status
kubectl describe cluster example-cluster -n default

# View all CAPI resources
kubectl get clusters,machines,machinedeployments -n default

# Check Rancher import status
kubectl get cluster.provisioning.cattle.io -n fleet-default
```

## Step 4: Validate in Rancher UI

1. Navigate to **Cluster Management** in Rancher
2. Verify the cluster appears in the list
3. Check cluster health indicators
4. Review node status and resource utilization

## Common Operations

```bash
# Scale worker nodes
kubectl scale machinedeployment example-cluster-workers --replicas=5

# Get cluster kubeconfig
clusterctl get kubeconfig example-cluster > cluster-kubeconfig.yaml

# Test connectivity
export KUBECONFIG=cluster-kubeconfig.yaml
kubectl get nodes

# Return to management cluster
unset KUBECONFIG
```

## Troubleshooting

```bash
# Check Turtles controller logs
kubectl logs -n rancher-turtles-system   -l control-plane=controller-manager   --follow

# Check CAPI controller logs
kubectl logs -n capi-system   -l control-plane=controller-manager   --since=30m

# Get events for a cluster
kubectl get events -n default   --field-selector involvedObject.name=example-cluster   --sort-by=.lastTimestamp
```

## Conclusion

How to Troubleshoot Rancher Turtles Cluster Provisioning with Rancher Turtles enables a declarative, Kubernetes-native approach to infrastructure management. By leveraging the Cluster API ecosystem alongside Rancher's management capabilities, you get a powerful, unified platform for managing Kubernetes clusters at scale across any infrastructure.
