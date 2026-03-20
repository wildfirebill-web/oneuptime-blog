# How to Deploy Linkerd with Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Linkerd, Service Mesh, mTLS

Description: Deploy and configure Linkerd service mesh on Rancher-managed clusters for automatic mTLS, observability, and traffic management between microservices.

## Introduction

Linkerd is a lightweight, open-source service mesh designed for simplicity and performance. Unlike heavier alternatives, Linkerd uses a Rust-based data plane proxy with minimal overhead. This guide covers installing Linkerd on a Rancher-managed cluster, enabling automatic mTLS, and setting up observability.

## Prerequisites

- Rancher-managed Kubernetes cluster (1.25+)
- kubectl configured for your cluster
- Helm 3.x installed
- `linkerd` CLI installed
- Cluster-admin permissions

## Step 1: Install the Linkerd CLI

```bash
# Install Linkerd CLI using the official install script

curl --proto '=https' --tlsv1.2 -sSfL https://run.linkerd.io/install | sh

# Add to PATH
export PATH=$PATH:$HOME/.linkerd2/bin

# Verify installation
linkerd version
```

## Step 2: Pre-Installation Checks

```bash
# Run Linkerd pre-install checks
linkerd check --pre

# Check Kubernetes version compatibility
kubectl version --short
```

## Step 3: Install Linkerd CRDs

```bash
# Add Linkerd Helm repository
helm repo add linkerd https://helm.linkerd.io/stable
helm repo update

# Install Linkerd CRDs first
helm install linkerd-crds linkerd/linkerd-crds \
  --namespace linkerd \
  --create-namespace
```

## Step 4: Install Linkerd Control Plane

```bash
# Generate certificates for Linkerd (production should use cert-manager or Vault)
step certificate create root.linkerd.cluster.local ca.crt ca.key \
  --profile root-ca \
  --no-password \
  --insecure

step certificate create identity.linkerd.cluster.local issuer.crt issuer.key \
  --profile intermediate-ca \
  --not-after 8760h \
  --no-password \
  --insecure \
  --ca ca.crt \
  --ca-key ca.key

# Install Linkerd control plane
helm install linkerd-control-plane linkerd/linkerd-control-plane \
  --namespace linkerd \
  --set-file identityTrustAnchorsPEM=ca.crt \
  --set-file identity.issuer.tls.crtPEM=issuer.crt \
  --set-file identity.issuer.tls.keyPEM=issuer.key
```

## Step 5: Verify Linkerd Installation

```bash
# Run post-install checks
linkerd check

# View Linkerd components
kubectl get pods -n linkerd

# Check Linkerd version running
linkerd version --client --short
```

## Step 6: Install Linkerd Viz Extension

```bash
# Install the Linkerd visualization dashboard
helm install linkerd-viz linkerd/linkerd-viz \
  --namespace linkerd-viz \
  --create-namespace

# Verify viz is running
linkerd viz check

# Open the dashboard
linkerd viz dashboard &
```

## Step 7: Inject Linkerd Proxy into Your Application

```yaml
# namespace-injection.yaml - Enable automatic injection for a namespace
apiVersion: v1
kind: Namespace
metadata:
  name: production
  annotations:
    # Enable automatic Linkerd proxy injection
    linkerd.io/inject: enabled
```

Or inject at the deployment level:

```yaml
# deployment.yaml - Deployment with Linkerd injection annotation
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: production
spec:
  template:
    metadata:
      annotations:
        # Enable Linkerd proxy sidecar injection
        linkerd.io/inject: enabled
    spec:
      containers:
        - name: my-app
          image: registry.example.com/my-app:v1.0
          ports:
            - containerPort: 8080
```

Apply and verify injection:

```bash
kubectl apply -f namespace-injection.yaml
kubectl rollout restart deployment/my-app -n production

# Verify proxy was injected
linkerd viz stat deployments -n production
```

## Step 8: Deploy a Sample Microservices Application

```bash
# Deploy Linkerd's demo bookinfo application
curl -sL https://run.linkerd.io/emojivoto.yml | kubectl apply -f -

# Inject Linkerd into the demo app
kubectl get -n emojivoto deploy -o yaml | \
  linkerd inject - | \
  kubectl apply -f -

# View traffic metrics
linkerd viz stat deployments -n emojivoto
```

## Step 9: Configure Traffic Policies

```yaml
# server-policy.yaml - Server authorization policy
apiVersion: policy.linkerd.io/v1beta1
kind: Server
metadata:
  name: my-app-server
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: my-app
  port: 8080
  proxyProtocol: HTTP/2
---
apiVersion: policy.linkerd.io/v1beta1
kind: ServerAuthorization
metadata:
  name: my-app-allow-frontend
  namespace: production
spec:
  server:
    name: my-app-server
  client:
    # Only allow access from the frontend service
    meshTLS:
      serviceAccounts:
        - name: frontend-sa
```

## Step 10: Monitor with Linkerd Viz

```bash
# View live traffic metrics
linkerd viz top deployment/my-app -n production

# View success rates and latency
linkerd viz stat deployments -n production --from deployment/frontend

# View routes
linkerd viz routes deployment/my-app -n production
```

## Conclusion

Linkerd provides a lightweight, production-grade service mesh that integrates well with Rancher-managed clusters. Its automatic mTLS, observability, and traffic management capabilities significantly improve microservice security and reliability with minimal configuration. The Linkerd CLI and visualization dashboard make it easy to monitor and troubleshoot inter-service communication, making it an excellent choice for teams prioritizing simplicity and performance.
