# How to Configure Kubewarden Validation Policies

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Kubewarden, Kubernetes, Policy, Validation, Security

Description: Learn how to configure Kubewarden validation policies to enforce security rules, compliance requirements, and operational standards by allowing or denying Kubernetes resource operations.

## Introduction

Validation policies are the most common type of Kubewarden policy. They evaluate incoming Kubernetes admission requests and either allow or deny them based on configurable rules. Unlike mutation policies that modify resources, validation policies are read-only — they simply approve or reject the admission request.

This guide covers creating comprehensive validation policies using both hub policies and custom implementations.

## Prerequisites

- Kubewarden installed on your Kubernetes cluster
- `kubectl` with cluster-admin access
- Basic understanding of Kubernetes admission control

## Core Validation Policy Settings

A validation policy `spec` contains these key fields:

```yaml
spec:
  module: registry://...       # Policy Wasm module location
  settings: {}                 # Policy-specific configuration
  mutating: false              # MUST be false for validation policies
  failurePolicy: Fail          # Fail or Ignore if policy errors
  mode: protect                # protect or monitor
  rules:                       # What requests to intercept
    - apiGroups: [...]
      apiVersions: [...]
      resources: [...]
      operations: [...]
```

## Comprehensive Security Validation Suite

### Policy 1: Block Privileged Containers

```yaml
# 01-no-privileged.yaml
apiVersion: policies.kubewarden.io/v1
kind: ClusterAdmissionPolicy
metadata:
  name: no-privileged-containers
  annotations:
    description: "Blocks pods with privileged containers"
    severity: "critical"
spec:
  module: registry://ghcr.io/kubewarden/policies/pod-privileged:v0.2.0
  rules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      resources: ["pods"]
      operations: ["CREATE", "UPDATE"]
  mutating: false
  failurePolicy: Fail
  mode: protect
```

### Policy 2: Require Non-Root User

```yaml
# 02-non-root.yaml
apiVersion: policies.kubewarden.io/v1
kind: ClusterAdmissionPolicy
metadata:
  name: require-non-root
spec:
  module: registry://ghcr.io/kubewarden/policies/user-group-psp:v0.4.0
  settings:
    run_as_user:
      rule: "MustRunAsNonRoot"
    run_as_group:
      rule: "MayRunAs"
      ranges:
        - min: 1000
          max: 65535
  rules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      resources: ["pods"]
      operations: ["CREATE", "UPDATE"]
  mutating: false
  failurePolicy: Fail
  mode: protect
```

### Policy 3: Block Host Path Volumes

```yaml
# 03-no-host-path.yaml
apiVersion: policies.kubewarden.io/v1
kind: ClusterAdmissionPolicy
metadata:
  name: no-host-path-volumes
spec:
  module: registry://ghcr.io/kubewarden/policies/volumes-psp:v0.2.0
  settings:
    allowedTypes:
      - configMap
      - secret
      - persistentVolumeClaim
      - emptyDir
      - projected
      # hostPath is intentionally omitted
  rules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      resources: ["pods"]
      operations: ["CREATE", "UPDATE"]
  mutating: false
  failurePolicy: Fail
  mode: protect
```

### Policy 4: Require Read-Only Root Filesystem

```yaml
# 04-readonly-rootfs.yaml
apiVersion: policies.kubewarden.io/v1
kind: ClusterAdmissionPolicy
metadata:
  name: require-readonly-rootfs
spec:
  module: registry://ghcr.io/kubewarden/policies/readonly-root-filesystem-psp:v0.2.0
  rules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      resources: ["pods"]
      operations: ["CREATE", "UPDATE"]
  mutating: false
  failurePolicy: Fail
  mode: protect
```

### Policy 5: Restrict Allowed Capabilities

```yaml
# 05-capabilities.yaml
apiVersion: policies.kubewarden.io/v1
kind: ClusterAdmissionPolicy
metadata:
  name: restrict-capabilities
spec:
  module: registry://ghcr.io/kubewarden/policies/capabilities-psp:v0.2.0
  settings:
    allowed_capabilities: []   # No additional capabilities
    required_drop_capabilities:
      - ALL                    # Drop all capabilities by default
  rules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      resources: ["pods"]
      operations: ["CREATE", "UPDATE"]
  mutating: false
  failurePolicy: Fail
  mode: protect
```

## Validation Policy for Ingress Resources

```yaml
# validate-ingress.yaml
apiVersion: policies.kubewarden.io/v1
kind: ClusterAdmissionPolicy
metadata:
  name: ingress-must-have-tls
spec:
  module: registry://ghcr.io/kubewarden/policies/ingress-policy:v0.1.0

  settings:
    # Require all ingresses to have TLS configured
    requireTLS: true
    # Allow only specific ingress classes
    allowedIngressClasses:
      - nginx
      - traefik

  rules:
    - apiGroups: ["networking.k8s.io"]
      apiVersions: ["v1"]
      resources: ["ingresses"]
      operations: ["CREATE", "UPDATE"]
  mutating: false
  failurePolicy: Fail
  mode: protect
```

## Namespace-Scoped Validation Policies

For per-team policies using AdmissionPolicy:

```yaml
# namespace-policy-team-backend.yaml
apiVersion: policies.kubewarden.io/v1
kind: AdmissionPolicy
metadata:
  name: backend-image-policy
  namespace: backend
spec:
  module: registry://ghcr.io/kubewarden/policies/allowed-image-repositories:v0.1.0

  settings:
    allowedRegistries:
      - registry.internal.example.com/backend

  rules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      resources: ["pods"]
      operations: ["CREATE", "UPDATE"]
  mutating: false
  failurePolicy: Fail
  mode: protect
```

## Applying Policies in Stages

Best practice is to apply policies in monitor mode first:

```bash
# Step 1: Apply all policies in monitor mode
find policies/ -name "*.yaml" -exec \
  sed 's/mode: protect/mode: monitor/' {} | \
  kubectl apply -f - \;

# Step 2: Monitor violations for a week
kubectl get events -A \
  --field-selector reason=PolicyViolation \
  --sort-by='.lastTimestamp' \
  -w

# Step 3: Fix non-compliant workloads

# Step 4: Switch to protect mode
kubectl get clusteradmissionpolicies \
  -o jsonpath='{.items[*].metadata.name}' | \
  tr ' ' '\n' | \
  xargs -I{} kubectl patch clusteradmissionpolicy {} \
    --type=merge \
    -p '{"spec":{"mode":"protect"}}'
```

## Verifying Policy Coverage

```bash
# List all active validation policies
kubectl get clusteradmissionpolicies -o wide

# Check which resources each policy covers
kubectl get clusteradmissionpolicies \
  -o jsonpath='{range .items[*]}{.metadata.name}: {.spec.rules[0].resources}{"\n"}{end}'

# Check for policy activation status
kubectl get clusteradmissionpolicies \
  -o jsonpath='{range .items[*]}{.metadata.name}: {.status.conditions[?(@.type=="PolicyActive")].status}{"\n"}{end}'
```

## Conclusion

Kubewarden validation policies form the core of your Kubernetes security enforcement layer. By deploying a comprehensive set of validation policies covering privileged access, host resources, volume types, and container security settings, you establish defense-in-depth against common Kubernetes security misconfigurations. The monitor-then-protect rollout strategy ensures you can safely introduce new policies without disrupting existing workloads, and namespace-scoped AdmissionPolicies let teams enforce additional standards relevant to their specific applications.
