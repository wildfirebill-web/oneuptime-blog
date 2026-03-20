# How to Install Elemental Operator

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Elemental, Kubernetes, Rancher, Operator, Edge

Description: A step-by-step guide to installing the Elemental Operator on a Rancher-managed Kubernetes cluster for edge node provisioning.

## Introduction

The Elemental Operator is the core component of SUSE's Elemental project, enabling declarative management of bare metal and edge nodes through Kubernetes-native APIs. It integrates with Rancher to provide a unified control plane for provisioning, registering, and managing OS images on remote machines.

This guide walks you through installing the Elemental Operator using Helm and verifying it is functioning correctly in your cluster.

## Prerequisites

Before installing the Elemental Operator, ensure you have:

- A running Rancher instance (v2.7 or later)
- `kubectl` configured to access your management cluster
- `helm` v3.x installed
- Cert-manager installed in the cluster

### Install Cert-Manager (if not already installed)

```bash
# Add the Jetstack Helm repository
helm repo add jetstack https://charts.jetstack.io
helm repo update

# Install cert-manager with CRDs
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true
```

## Adding the Elemental Helm Repository

```bash
# Add the Rancher Elemental Helm chart repository
helm repo add elemental-operator https://rancher.github.io/elemental-operator/

# Update the local chart index
helm repo update

# Verify the repo was added
helm search repo elemental-operator
```

## Installing the Elemental Operator

### Install with Default Settings

```bash
# Create the elemental-system namespace
kubectl create namespace elemental-system

# Install the Elemental Operator
helm install elemental-operator elemental-operator/elemental-operator \
  --namespace elemental-system
```

### Install with Custom Values

For production environments, you may want to customize the installation:

```bash
# Install with custom values
helm install elemental-operator elemental-operator/elemental-operator \
  --namespace elemental-system \
  --set image.tag=latest \
  --set replicas=2 \
  --set resources.requests.cpu=100m \
  --set resources.requests.memory=128Mi
```

Alternatively, create a `values.yaml` file:

```yaml
# elemental-values.yaml
replicas: 2

image:
  repository: registry.suse.com/rancher/elemental-operator
  tag: "1.4.0"

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi

# Enable webhook for validation
webhook:
  enabled: true
```

```bash
helm install elemental-operator elemental-operator/elemental-operator \
  --namespace elemental-system \
  --values elemental-values.yaml
```

## Verifying the Installation

```bash
# Check that the operator pod is running
kubectl get pods -n elemental-system

# Expected output:
# NAME                                   READY   STATUS    RESTARTS   AGE
# elemental-operator-7d9f8b6c5-xpqrt     1/1     Running   0          2m

# Check the CRDs were installed
kubectl get crd | grep elemental

# Verify the operator is ready
kubectl rollout status deployment elemental-operator -n elemental-system
```

## Verifying CRDs

The Elemental Operator installs several Custom Resource Definitions:

```bash
# List Elemental CRDs
kubectl get crd | grep elemental.cattle.io
```

You should see resources including:
- `machineregistrations.elemental.cattle.io`
- `machineinventories.elemental.cattle.io`
- `machineinventoryselectors.elemental.cattle.io`
- `machineinventoryselectortemplate.elemental.cattle.io`
- `managedosversionchannels.elemental.cattle.io`
- `managedosversions.elemental.cattle.io`
- `managedosimages.elemental.cattle.io`

## Checking Operator Logs

```bash
# Stream operator logs to verify it started correctly
kubectl logs -n elemental-system \
  -l app=elemental-operator \
  --follow
```

## Upgrading the Elemental Operator

To upgrade to a newer version:

```bash
# Update the Helm repo
helm repo update

# Upgrade the operator
helm upgrade elemental-operator elemental-operator/elemental-operator \
  --namespace elemental-system \
  --reuse-values
```

## Uninstalling the Elemental Operator

```bash
# Remove the operator
helm uninstall elemental-operator -n elemental-system

# Optionally remove CRDs (warning: this deletes all Elemental resources)
kubectl get crd | grep elemental.cattle.io | awk '{print $1}' | xargs kubectl delete crd
```

## Conclusion

The Elemental Operator provides the Kubernetes-native foundation for managing bare metal and edge devices at scale. Once installed, you can proceed to create MachineRegistrations, build OS images, and begin provisioning nodes. The operator's declarative model ensures consistent and reproducible infrastructure across your edge fleet.
