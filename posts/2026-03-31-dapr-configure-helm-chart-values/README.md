# How to Configure Dapr Helm Chart Values on Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Helm, Kubernetes, Configuration, Installation

Description: Master the key Dapr Helm chart values for controlling HA mode, resource limits, image versions, mTLS, and observability during installation and upgrades.

---

## Discovering Available Values

Before customizing, explore the full list of configurable values:

```bash
helm repo add dapr https://dapr.github.io/helm-charts/
helm repo update

# Show all available values
helm show values dapr/dapr > dapr-default-values.yaml

# Show values for a specific version
helm show values dapr/dapr --version 1.13.0
```

## Core Global Values

```yaml
# values.yaml
global:
  # Enable high availability for all components
  ha:
    enabled: true
    replicaCount: 3

  # Set log level for all components
  logLevel: info

  # Set log format
  logAsJson: true

  # Enable mTLS between services
  mtls:
    enabled: true
    workloadCertTTL: "24h"
    allowedClockSkew: "15m"

  # Registry for Dapr images (override for air-gapped installs)
  registry: docker.io/daprio
  tag: "1.13.0"
```

## Component-Specific Values

```yaml
dapr_operator:
  replicaCount: 2
  logLevel: info
  resources:
    requests:
      cpu: "100m"
      memory: "256Mi"
    limits:
      cpu: "500m"
      memory: "512Mi"
  nodeSelector:
    kubernetes.io/os: linux

dapr_sentry:
  replicaCount: 2
  logLevel: info
  resources:
    requests:
      cpu: "100m"
      memory: "128Mi"
    limits:
      cpu: "300m"
      memory: "256Mi"

dapr_placement:
  replicaCount: 3
  logLevel: info
  resources:
    requests:
      cpu: "250m"
      memory: "256Mi"
    limits:
      cpu: "1000m"
      memory: "512Mi"

dapr_sidecar_injector:
  replicaCount: 2
  logLevel: info
  resources:
    requests:
      cpu: "100m"
      memory: "128Mi"
    limits:
      cpu: "300m"
      memory: "256Mi"

dapr_dashboard:
  enabled: true
  replicaCount: 1
  logLevel: info
```

## Installing with Custom Values

```bash
helm install dapr dapr/dapr \
  --namespace dapr-system \
  --create-namespace \
  --version 1.13.0 \
  -f values.yaml \
  --wait

# Or with --set flags for individual values
helm install dapr dapr/dapr \
  --namespace dapr-system \
  --set global.ha.enabled=true \
  --set global.logAsJson=true \
  --set dapr_operator.replicaCount=2
```

## Upgrading with New Values

```bash
# Upgrade preserving existing values + adding new ones
helm upgrade dapr dapr/dapr \
  --namespace dapr-system \
  --version 1.14.0 \
  -f values.yaml \
  --reuse-values \
  --wait
```

## Viewing Deployed Values

```bash
# See what values are currently deployed
helm get values dapr -n dapr-system

# See all values (including defaults)
helm get values dapr -n dapr-system --all
```

## Summary

Managing Dapr through Helm values files provides version-controlled, reproducible deployments. Store your values.yaml in a GitOps repository and use `helm upgrade --reuse-values` to preserve existing settings when upgrading the Dapr version. The `global.ha.enabled=true` value is the single most important production setting.
