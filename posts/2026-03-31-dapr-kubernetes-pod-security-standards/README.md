# How to Use Dapr with Kubernetes Pod Security Standards

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Kubernetes, Pod Security, Security, Compliance

Description: Configure Dapr-enabled pods to comply with Kubernetes Pod Security Standards including baseline and restricted profiles while maintaining full Dapr sidecar functionality.

---

## Overview

Kubernetes Pod Security Standards (PSS) replaced PodSecurityPolicies and define three security profiles: privileged, baseline, and restricted. Running Dapr with the restricted profile requires careful configuration of both the application and the Dapr sidecar injector.

## Labeling Namespaces with Pod Security Standards

Apply the baseline security standard to a Dapr namespace:

```bash
# Apply baseline enforcement (recommended for most Dapr workloads)
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/warn=restricted \
  pod-security.kubernetes.io/audit=restricted
```

## Configuring Dapr for Restricted Profile

The restricted profile requires non-root users and read-only root filesystems. Configure Dapr annotations accordingly:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: production
spec:
  template:
    metadata:
      annotations:
        dapr.io/enabled: "true"
        dapr.io/app-id: "order-service"
        dapr.io/app-port: "3000"
        dapr.io/sidecar-seccomp-profile-type: "RuntimeDefault"
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        seccompProfile:
          type: RuntimeDefault
      containers:
      - name: order-service
        image: myregistry/order-service:latest
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
            - ALL
```

## Dapr Operator Configuration for Restricted Namespaces

Ensure the Dapr operator injects sidecars with compliant security contexts. Patch the Dapr operator config:

```bash
# Check current Dapr operator security context
kubectl get deployment dapr-operator -n dapr-system -o yaml | grep -A 20 securityContext
```

Update the Dapr Helm values to use restricted-compatible settings:

```yaml
# values.yaml for dapr helm chart
global:
  securityContext:
    runAsNonRoot: true

dapr_operator:
  replicaCount: 2
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
```

```bash
helm upgrade dapr dapr/dapr -n dapr-system -f values.yaml
```

## Testing Pod Security Compliance

Dry-run a deployment to check for PSS violations before applying:

```bash
kubectl apply --dry-run=server -f order-service.yaml
# Output: Warning: would violate PodSecurity "restricted:latest":
# allowPrivilegeEscalation != false (container "daprd")
```

## Exempting the Dapr System Namespace

The dapr-system namespace needs exemptions for the operator and placement service:

```bash
kubectl label namespace dapr-system \
  pod-security.kubernetes.io/enforce=privileged
```

## Auditing Existing Pods

Find non-compliant pods in a Dapr namespace:

```bash
kubectl label namespace production \
  pod-security.kubernetes.io/audit=restricted --overwrite

# Check audit events
kubectl get events -n production | grep PodSecurity
```

## Summary

Dapr works with Kubernetes Pod Security Standards when you configure proper security contexts on both your application containers and the Dapr sidecar injector. Use the baseline profile for most Dapr workloads and apply restricted settings per-deployment where compliance requirements demand it. Always exempt the dapr-system namespace itself, as its components require elevated privileges to function.
