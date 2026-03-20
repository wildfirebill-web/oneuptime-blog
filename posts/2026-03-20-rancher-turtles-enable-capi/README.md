# How to Enable Cluster API with Rancher Turtles

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher Turtles, CAPI, Kubernetes, Rancher, Infrastructure

Description: Learn how to enable and configure Cluster API providers in Rancher Turtles for managing Kubernetes cluster infrastructure across multiple platforms.

## Introduction

Cluster API (CAPI) is a Kubernetes sub-project that provides declarative APIs for cluster lifecycle management. Rancher Turtles integrates CAPI into Rancher, enabling you to use CAPI's provider ecosystem while managing clusters through Rancher's unified interface.

## Understanding CAPI Components

CAPI has four main component types:

| Component | Function |
|-----------|----------|
| Core Provider | Core CAPI controllers and APIs |
| Bootstrap Provider | Configures nodes to join clusters (cloud-init, RKE2) |
| Control Plane Provider | Manages control plane lifecycle |
| Infrastructure Provider | Creates cloud infrastructure (AWS, Azure, vSphere) |

## Enabling CAPI Providers via CAPIProvider CRD

Rancher Turtles introduces a `CAPIProvider` CRD for managing providers:

```yaml
# enable-aws-provider.yaml
apiVersion: turtles-capi.cattle.io/v1alpha1
kind: CAPIProvider
metadata:
  name: aws
  namespace: rancher-turtles-system
spec:
  name: aws
  type: infrastructure
  version: v2.3.0
  configSecret:
    name: aws-credentials
```

```bash
kubectl apply -f enable-aws-provider.yaml

# Verify provider is installed
kubectl get capiprovider -n rancher-turtles-system
```

## Installing the CAPI Operator

The CAPI Operator manages provider installations:

```bash
# Install using clusterctl
clusterctl init \
  --infrastructure aws \
  --bootstrap rke2 \
  --control-plane rke2

# Verify all providers are installed
clusterctl describe

# Check provider versions
kubectl get providers -A
```

## Configuring the RKE2 Bootstrap Provider

```yaml
# rke2-bootstrap-provider.yaml
apiVersion: turtles-capi.cattle.io/v1alpha1
kind: CAPIProvider
metadata:
  name: rke2-bootstrap
  namespace: rancher-turtles-system
spec:
  name: rke2
  type: bootstrap
  version: v0.3.0
  variables:
    # RKE2 version to use for bootstrapped clusters
    RKE2_VERSION: "v1.28.0+rke2r1"
```

## Enabling Auto-Import for CAPI Clusters

Configure Rancher Turtles to automatically import CAPI clusters into Rancher:

```bash
# Label the namespace where CAPI clusters are created
kubectl label namespace default \
  cluster-api.cattle.io/rancher-auto-import=true

# Or enable globally in Turtles configuration
kubectl patch configmap rancher-turtles-config \
  -n rancher-turtles-system \
  --type merge \
  -p '{"data":{"auto-import":"true"}}'
```

## Verifying Provider Status

```bash
# List all installed providers
kubectl get providers -A -o wide

# Check provider health
kubectl describe provider capi-system/cluster-api

# Watch provider bootstrap
kubectl get providers -A --watch

# Check CAPI controller logs
kubectl logs -n capi-system \
  -l control-plane=controller-manager \
  --follow
```

## Configuring CAPI Variables

Set environment variables required by infrastructure providers:

```bash
# For AWS
export AWS_REGION=us-west-2
export AWS_ACCESS_KEY_ID=<your-key>
export AWS_SECRET_ACCESS_KEY=<your-secret>

# Create credentials secret
kubectl create secret generic aws-credentials \
  --namespace capa-system \
  --from-literal=AWS_REGION=${AWS_REGION} \
  --from-literal=AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID} \
  --from-literal=AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}
```

## Conclusion

Enabling Cluster API with Rancher Turtles extends Rancher's cluster management capabilities to the full CAPI provider ecosystem. By installing infrastructure, bootstrap, and control plane providers, you gain the ability to provision Kubernetes clusters declaratively across any platform that has a CAPI provider, all managed through Rancher's unified interface.
