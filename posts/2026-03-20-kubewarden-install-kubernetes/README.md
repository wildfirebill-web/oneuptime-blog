# How to Install Kubewarden on Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Kubewarden, Kubernetes, Policy, Security, Admission Control

Description: A step-by-step guide to installing Kubewarden on a Kubernetes cluster using Helm, including CRDs, the policy server, and initial configuration.

## Introduction

Kubewarden is a Kubernetes admission controller that uses WebAssembly (Wasm) policies to validate and mutate Kubernetes resources. Unlike traditional admission controllers that require writing code in specific languages, Kubewarden supports policies written in Rust, Go, Python, Swift, and any language that compiles to WebAssembly.

This guide covers the complete installation of Kubewarden on a Kubernetes cluster from scratch.

## Prerequisites

- Kubernetes cluster v1.24 or later
- Helm v3.8 or later
- `kubectl` configured with cluster access
- Cluster-admin permissions
- Cert-manager (required for Kubewarden webhooks)

## Architecture Overview

Kubewarden consists of:
- **kubewarden-controller**: Manages PolicyServer and AdmissionPolicy lifecycle
- **PolicyServer**: Runs WebAssembly policies and acts as the webhook server
- **AdmissionPolicy**: Namespace-scoped policy definitions
- **ClusterAdmissionPolicy**: Cluster-scoped policy definitions

## Step 1: Install Cert-Manager

Kubewarden requires cert-manager for managing TLS certificates:

```bash
# Add the cert-manager Helm repository

helm repo add jetstack https://charts.jetstack.io
helm repo update

# Install cert-manager with CRDs
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true \
  --version v1.14.0

# Verify cert-manager is running
kubectl get pods -n cert-manager
```

Wait until all cert-manager pods are in `Running` state before proceeding.

## Step 2: Add the Kubewarden Helm Repository

```bash
# Add Kubewarden Helm charts repository
helm repo add kubewarden https://charts.kubewarden.io
helm repo update

# List available Kubewarden charts
helm search repo kubewarden
```

## Step 3: Install Kubewarden CRDs

```bash
# Create the kubewarden-system namespace
kubectl create namespace kubewarden

# Install Kubewarden Custom Resource Definitions
helm install kubewarden-crds kubewarden/kubewarden-crds \
  --namespace kubewarden \
  --wait

# Verify CRDs are installed
kubectl get crds | grep kubewarden
```

Expected output includes:
```text
admissionpolicies.policies.kubewarden.io
clusteradmissionpolicies.policies.kubewarden.io
policyservers.policies.kubewarden.io
```

## Step 4: Install the Kubewarden Controller

```bash
# Install the Kubewarden controller
helm install kubewarden-controller kubewarden/kubewarden-controller \
  --namespace kubewarden \
  --wait

# Verify the controller is running
kubectl get pods -n kubewarden
```

## Step 5: Install the Default Policy Server

```bash
# Install the default Kubewarden policy server
helm install kubewarden-defaults kubewarden/kubewarden-defaults \
  --namespace kubewarden \
  --wait

# Verify the policy server is running
kubectl get pods -n kubewarden
kubectl get policyserver
```

Expected output:
```text
NAME      AGE
default   1m
```

## Step 6: Verify the Complete Installation

```bash
# Check all Kubewarden components are running
kubectl get all -n kubewarden

# Verify the policy server is active
kubectl describe policyserver default

# Check the webhook configuration
kubectl get validatingwebhookconfigurations \
  | grep kubewarden

kubectl get mutatingwebhookconfigurations \
  | grep kubewarden

# Run a test to ensure the admission webhook is working
kubectl apply --dry-run=server -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
  namespace: default
spec:
  containers:
    - name: test
      image: nginx:latest
EOF
```

## Installing with Custom Options

### Production Installation with Custom Resources

```bash
# Install with production-ready resource settings
helm install kubewarden-controller kubewarden/kubewarden-controller \
  --namespace kubewarden \
  --set controller.resources.requests.cpu="100m" \
  --set controller.resources.requests.memory="128Mi" \
  --set controller.resources.limits.cpu="500m" \
  --set controller.resources.limits.memory="512Mi" \
  --set controller.replicaCount=2

# Install policy server with HA
helm install kubewarden-defaults kubewarden/kubewarden-defaults \
  --namespace kubewarden \
  --set policyServer.replicaCount=3
```

### Air-Gapped Installation

For air-gapped environments, pre-pull all images:

```bash
# Get the list of images needed
helm template kubewarden-controller kubewarden/kubewarden-controller \
  | grep "image:" \
  | sort -u

# Pull and push images to your private registry
# Then install with custom image registry
helm install kubewarden-controller kubewarden/kubewarden-controller \
  --namespace kubewarden \
  --set global.imageRegistry=registry.internal.example.com
```

## Upgrading Kubewarden

```bash
# Update the Helm repo
helm repo update

# Upgrade CRDs first
helm upgrade kubewarden-crds kubewarden/kubewarden-crds \
  --namespace kubewarden

# Upgrade the controller
helm upgrade kubewarden-controller kubewarden/kubewarden-controller \
  --namespace kubewarden

# Upgrade the defaults (policy server)
helm upgrade kubewarden-defaults kubewarden/kubewarden-defaults \
  --namespace kubewarden
```

## Uninstalling Kubewarden

```bash
# Remove in reverse order
helm uninstall kubewarden-defaults -n kubewarden
helm uninstall kubewarden-controller -n kubewarden
helm uninstall kubewarden-crds -n kubewarden

# Delete the namespace
kubectl delete namespace kubewarden
```

## Conclusion

Installing Kubewarden on Kubernetes sets up a powerful, WebAssembly-based admission control system. The three-step Helm installation - CRDs, controller, and policy server - provides a clean, upgradeable installation that integrates with cert-manager for automatic certificate management. With Kubewarden installed, you are ready to deploy admission policies that enforce security, compliance, and operational best practices across your cluster.
