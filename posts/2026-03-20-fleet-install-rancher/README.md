# How to Install Fleet in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Fleet, GitOps, Rancher, Kubernetes, Continuous Delivery

Description: A step-by-step guide to installing and enabling Fleet in Rancher for GitOps-based continuous delivery across multiple Kubernetes clusters.

## Introduction

Fleet is Rancher's built-in GitOps continuous delivery tool that enables you to manage Kubernetes resources across multiple clusters from a single Git repository. When you install Rancher, Fleet is included by default, but understanding how to configure and enable it properly is essential for production use.

This guide walks you through installing and setting up Fleet within Rancher, covering both the Rancher UI approach and the Helm-based installation for standalone Fleet deployments.

## Prerequisites

Before installing Fleet, ensure you have:

- A running Rancher instance (v2.6 or later)
- Kubernetes clusters registered in Rancher
- `kubectl` configured with access to your local cluster
- Helm v3 installed on your workstation
- Sufficient RBAC permissions in Rancher

## Fleet Architecture Overview

Fleet operates with two primary components:

- **Fleet Manager**: Runs in the Rancher local cluster and coordinates deployments
- **Fleet Agent**: Runs in each downstream cluster and applies resources

## Method 1: Enabling Fleet via Rancher UI

Fleet is bundled with Rancher and can be enabled directly from the UI.

### Step 1: Navigate to Continuous Delivery

1. Log in to your Rancher dashboard
2. Click the **hamburger menu** (top-left)
3. Select **Continuous Delivery**

If Fleet is not yet activated, Rancher will prompt you to enable it.

### Step 2: Configure Fleet Settings

Navigate to **Continuous Delivery > Advanced > Settings** to configure:

- Default workspace behavior
- Git polling intervals
- Agent deployment options

## Method 2: Installing Fleet Standalone with Helm

For environments where you want Fleet outside of Rancher, install it directly via Helm.

### Step 1: Add the Fleet Helm Repository

```bash
# Add the Rancher Fleet Helm chart repository
helm repo add fleet https://rancher.github.io/fleet-helm-charts/

# Update your local Helm chart repository cache
helm repo update
```

### Step 2: Install Fleet CRDs

Fleet requires Custom Resource Definitions to be installed first:

```bash
# Install Fleet CRDs into the cluster
helm install fleet-crd fleet/fleet-crd \
  --namespace cattle-fleet-system \
  --create-namespace
```

### Step 3: Install the Fleet Manager

```bash
# Install the Fleet controller/manager
helm install fleet fleet/fleet \
  --namespace cattle-fleet-system \
  --create-namespace \
  --set apiServerURL="https://your-kubernetes-api-server:6443"
```

### Step 4: Verify Fleet Installation

```bash
# Check that Fleet pods are running
kubectl get pods -n cattle-fleet-system

# Verify Fleet CRDs are installed
kubectl get crds | grep fleet

# Check Fleet cluster status
kubectl get clusters.fleet.cattle.io -A
```

Expected output should show Fleet manager pods in `Running` state:

```
NAME                                    READY   STATUS    RESTARTS   AGE
fleet-controller-7d9b8c6f4-x2pqr       1/1     Running   0          2m
fleet-gitjob-5c8b9f7d6-k4mnp           1/1     Running   0          2m
```

## Method 3: Installing Fleet Agent on Downstream Clusters

If you are managing remote clusters, install the Fleet agent on each downstream cluster.

### Step 1: Get the Fleet Manager URL and Token

```bash
# Retrieve the Fleet manager server URL
kubectl get secret -n cattle-fleet-system fleet-controller-bootstrap-token \
  -o jsonpath='{.data.token}' | base64 -d
```

### Step 2: Install the Agent

```bash
# Install Fleet agent on the downstream cluster
# Replace API_SERVER_URL and FLEET_TOKEN with your values
helm install fleet-agent fleet/fleet-agent \
  --namespace cattle-fleet-system \
  --create-namespace \
  --set apiServerURL="https://fleet-manager.example.com" \
  --set token="<FLEET_TOKEN>" \
  --set clusterNamespace="fleet-default"
```

## Configuring Fleet After Installation

### Setting the System Namespace

Fleet uses the `cattle-fleet-system` namespace by default. You can verify:

```bash
# List all Fleet-related namespaces
kubectl get namespaces | grep cattle-fleet
```

### Enabling Fleet in Rancher Feature Flags

For older Rancher versions, enable Fleet via the feature flags API:

```bash
# Patch the Fleet feature flag to enable it
kubectl patch feature fleet \
  --type merge \
  -p '{"spec": {"value": true}}'
```

## Verifying the Complete Installation

Run these checks to confirm Fleet is operational:

```bash
# Check Fleet bundle deployments
kubectl get bundles -A

# View GitRepo objects (should be empty until you create them)
kubectl get gitrepos -A

# Verify cluster registration
kubectl get clusters.fleet.cattle.io -n fleet-local
```

## Troubleshooting Common Installation Issues

### Fleet Pods Not Starting

If Fleet pods fail to start, check events and logs:

```bash
# Check pod events
kubectl describe pod -n cattle-fleet-system -l app=fleet-controller

# View Fleet controller logs
kubectl logs -n cattle-fleet-system -l app=fleet-controller
```

### Certificate Issues

If you encounter TLS errors:

```bash
# Verify the Fleet CA bundle
kubectl get secret -n cattle-fleet-system fleet-controller-bootstrap-token -o yaml
```

## Conclusion

Installing Fleet in Rancher provides a powerful GitOps foundation for managing Kubernetes workloads at scale. Whether you use the built-in Rancher UI or deploy Fleet standalone via Helm, the result is a robust continuous delivery platform capable of synchronizing applications across dozens or hundreds of clusters simultaneously. With Fleet installed, you are ready to configure Git repositories and start managing workloads declaratively.
