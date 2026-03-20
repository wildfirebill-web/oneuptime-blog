# How to Install Rancher Turtles

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher Turtles, CAPI, Kubernetes, Rancher, Cluster Management

Description: A step-by-step guide to installing Rancher Turtles, the Cluster API integration for Rancher that enables declarative Kubernetes cluster lifecycle management.

## Introduction

Rancher Turtles is the official Cluster API (CAPI) integration for Rancher. It enables Rancher to manage Kubernetes cluster lifecycle using the Cluster API framework, providing a consistent interface for creating and managing clusters across multiple cloud providers, on-premises, and edge environments.

## Prerequisites

- Rancher v2.9+ installed and accessible
- `kubectl` configured for your management cluster
- `helm` v3.x installed
- cert-manager installed in the cluster

## Step 1: Install cert-manager

```bash
# Add Jetstack Helm repo
helm repo add jetstack https://charts.jetstack.io
helm repo update

# Install cert-manager
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true

# Verify installation
kubectl get pods -n cert-manager
```

## Step 2: Add the Rancher Turtles Helm Repository

```bash
# Add the Rancher Turtles Helm repo
helm repo add turtles https://rancher.github.io/turtles
helm repo update

# Verify the repo is available
helm search repo turtles
```

## Step 3: Install Rancher Turtles

```bash
# Create the namespace
kubectl create namespace rancher-turtles-system

# Install Rancher Turtles
helm install rancher-turtles turtles/rancher-turtles \
  --namespace rancher-turtles-system \
  --dependency-update
```

### Install with Custom Values

```yaml
# turtles-values.yaml
# Cluster API operator settings
cluster-api-operator:
  enabled: true
  cluster-api:
    enabled: true
    version: v1.6.0
    core:
      namespace: capi-system
    bootstrap:
      rke2:
        enabled: true
        version: v0.3.0
        namespace: rke2-bootstrap-system
    controlPlane:
      rke2:
        enabled: true
        version: v0.3.0
        namespace: rke2-control-plane-system

# Turtles operator settings
rancher-turtles:
  imagePullPolicy: IfNotPresent
```

```bash
helm install rancher-turtles turtles/rancher-turtles \
  --namespace rancher-turtles-system \
  --values turtles-values.yaml \
  --dependency-update
```

## Step 4: Verify the Installation

```bash
# Check all Turtles pods are running
kubectl get pods -n rancher-turtles-system

# Check CAPI pods
kubectl get pods -n capi-system
kubectl get pods -n rke2-bootstrap-system
kubectl get pods -n rke2-control-plane-system

# Verify CRDs are installed
kubectl get crd | grep cluster.x-k8s.io

# Check that Turtles controller is running
kubectl rollout status deployment \
  rancher-turtles-controller-manager \
  -n rancher-turtles-system
```

## Step 5: Enable CAPI in Rancher UI

1. Log into Rancher
2. Navigate to **Home** > **Cluster Management**
3. Check that the CAPI integration appears in the menu
4. Verify the Cluster API Dashboard is accessible

## Step 6: Install Infrastructure Providers

After installing Turtles, install the infrastructure providers you need:

```bash
# Install AWS infrastructure provider
kubectl apply -f https://github.com/kubernetes-sigs/cluster-api-provider-aws/releases/latest/download/infrastructure-components.yaml

# Or using clusterctl
clusterctl init --infrastructure aws
```

## Upgrading Rancher Turtles

```bash
# Update the Helm repo
helm repo update

# Upgrade Turtles
helm upgrade rancher-turtles turtles/rancher-turtles \
  --namespace rancher-turtles-system \
  --reuse-values
```

## Uninstalling Rancher Turtles

```bash
# Remove the Helm release
helm uninstall rancher-turtles -n rancher-turtles-system

# Remove the namespace
kubectl delete namespace rancher-turtles-system

# Optionally remove CRDs (warning: deletes all CAPI resources)
kubectl get crd | grep cluster.x-k8s.io | \
  awk '{print $1}' | xargs kubectl delete crd
```

## Conclusion

Rancher Turtles bridges the Cluster API ecosystem with Rancher's management capabilities, giving you a unified interface for managing clusters across any infrastructure. Once installed, you can use CAPI providers to provision clusters on AWS, Azure, vSphere, and more, with all cluster resources visible and manageable through the Rancher UI.
