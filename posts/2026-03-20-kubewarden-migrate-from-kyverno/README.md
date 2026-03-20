# How to Migrate from Kyverno to Kubewarden

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Kubewarden, Kubernetes, Kyverno, Migration, Policy

Description: A practical guide to migrating your Kubernetes admission control policies from Kyverno to Kubewarden, including policy mapping, side-by-side testing, and cutover strategies.

## Introduction

Kyverno and Kubewarden both provide Kubernetes admission control, but with different approaches. Kyverno uses YAML-based policy definitions with a Kubernetes-native DSL, while Kubewarden uses WebAssembly policies that offer full programming language flexibility. Organizations migrate to Kubewarden for more complex validation logic, better performance, or to leverage existing programming skills.

This guide covers migrating from Kyverno to Kubewarden with a structured, low-risk approach.

## Prerequisites

- Existing Kyverno installation
- Kubewarden installed (can run alongside Kyverno initially)
- `kubectl` with cluster-admin access
- Inventory of existing Kyverno policies

## Understanding the Differences

| Feature | Kyverno | Kubewarden |
|---------|---------|------------|
| Policy definition | YAML/JMESPath | WebAssembly modules |
| Mutation | Native YAML patches | JSON patches from code |
| Generate | Yes (create resources) | No (validation/mutation only) |
| Language | YAML/CEL/JMESPath | Rust, Go, Any Wasm |
| Testing | Kyverno CLI | kwctl |
| Context | Native K8s API lookups | Host capability calls |

## Step 1: Inventory Existing Kyverno Policies

```bash
# List all Kyverno policies

kubectl get policies -A
kubectl get clusterpolicies

# Export all cluster policies
kubectl get clusterpolicies -o yaml > kyverno-clusterpolicies.yaml

# Export all namespace policies
kubectl get policies -A -o yaml > kyverno-policies.yaml

# Get a count of each policy type
echo "ClusterPolicies: $(kubectl get clusterpolicies --no-headers | wc -l)"
echo "Namespace Policies: $(kubectl get policies -A --no-headers | wc -l)"
```

## Step 2: Map Kyverno Policies to Kubewarden

### Disallow Privileged Containers

**Kyverno version:**
```yaml
# kyverno-disallow-privileged.yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-privileged-containers
spec:
  validationFailureAction: Enforce
  rules:
    - name: check-privileged
      match:
        any:
          - resources:
              kinds: [Pod]
      validate:
        message: "Privileged mode is disallowed."
        pattern:
          spec:
            containers:
              - =(securityContext):
                  =(privileged): "false"
```

**Kubewarden equivalent:**
```yaml
# kubewarden-disallow-privileged.yaml
apiVersion: policies.kubewarden.io/v1
kind: ClusterAdmissionPolicy
metadata:
  name: disallow-privileged-containers
spec:
  module: registry://ghcr.io/kubewarden/policies/pod-privileged:v0.2.0
  rules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      resources: ["pods"]
      operations: ["CREATE", "UPDATE"]
  mutating: false
  mode: protect
```

### Require Resource Limits

**Kyverno version:**
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-resources
spec:
  validationFailureAction: Enforce
  rules:
    - name: check-container-resources
      match:
        any:
          - resources:
              kinds: [Pod]
      validate:
        message: "CPU and memory limits are required."
        pattern:
          spec:
            containers:
              - resources:
                  limits:
                    memory: "?*"
                    cpu: "?*"
```

**Kubewarden equivalent:**
```yaml
apiVersion: policies.kubewarden.io/v1
kind: ClusterAdmissionPolicy
metadata:
  name: require-resource-limits
spec:
  module: registry://ghcr.io/kubewarden/policies/require-resources:v0.1.0
  settings:
    memory:
      requireLimit: true
    cpu:
      requireLimit: true
  rules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      resources: ["pods"]
      operations: ["CREATE", "UPDATE"]
  mutating: false
  mode: protect
```

### Require Labels

**Kyverno version:**
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-labels
spec:
  validationFailureAction: Enforce
  rules:
    - name: check-for-labels
      match:
        any:
          - resources:
              kinds: [Pod]
      validate:
        message: "The label `app` is required."
        pattern:
          metadata:
            labels:
              app: "?*"
```

**Kubewarden equivalent:**
```yaml
apiVersion: policies.kubewarden.io/v1
kind: ClusterAdmissionPolicy
metadata:
  name: require-app-label
spec:
  module: registry://ghcr.io/kubewarden/policies/k8s-objects:v1.3.0
  settings:
    mandatory_labels:
      - app
  rules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      resources: ["pods"]
      operations: ["CREATE", "UPDATE"]
  mutating: false
  mode: protect
```

### Image Registry Restriction

**Kyverno version:**
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: restrict-image-registries
spec:
  validationFailureAction: Enforce
  rules:
    - name: validate-registries
      match:
        any:
          - resources:
              kinds: [Pod]
      validate:
        message: "Unknown image registry."
        pattern:
          spec:
            containers:
              - image: "registry.internal.example.com/* | gcr.io/my-org/*"
```

**Kubewarden equivalent:**
```yaml
apiVersion: policies.kubewarden.io/v1
kind: ClusterAdmissionPolicy
metadata:
  name: restrict-image-registries
spec:
  module: registry://ghcr.io/kubewarden/policies/allowed-image-repositories:v0.1.0
  settings:
    allowedRegistries:
      - registry.internal.example.com
      - gcr.io/my-org
  rules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      resources: ["pods"]
      operations: ["CREATE", "UPDATE"]
  mutating: false
  mode: protect
```

## Step 3: Handle Kyverno Mutation Policies

Kyverno's mutation policies need to be rewritten as Kubewarden mutation policies in Wasm:

```yaml
# Kyverno mutation (adds default labels)
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: add-labels
spec:
  rules:
    - name: add-app-label
      match:
        any:
          - resources:
              kinds: [Pod]
      mutate:
        patchStrategicMerge:
          metadata:
            labels:
              managed-by: kubewarden
```

For mutation policies, you need to write a Kubewarden Wasm policy. The migration approach:
1. Check if the mutation is available on the Kubewarden Policy Hub
2. Write a custom Wasm policy if not available
3. Test extensively with `kwctl`

## Step 4: Side-by-Side Migration

```bash
#!/bin/bash
# migrate-policy.sh - Migrate one Kyverno policy to Kubewarden

KYVERNO_POLICY="$1"
KUBEWARDEN_POLICY="$2"

# Step 1: Deploy Kubewarden policy in monitor mode
kubectl apply -f "${KUBEWARDEN_POLICY}"
kubectl patch clusteradmissionpolicy \
  $(grep "name:" "${KUBEWARDEN_POLICY}" | head -1 | awk '{print $2}') \
  --type=merge \
  -p '{"spec":{"mode":"monitor"}}'

echo "Kubewarden policy deployed in monitor mode"

# Step 2: Compare violations for 24 hours
echo "Monitor for 24 hours and compare violations..."

# Check Kyverno violations
echo "=== Kyverno Violations ==="
kubectl get events -A \
  --field-selector reason=PolicyViolation \
  | grep kyverno

# Check Kubewarden monitor events
echo "=== Kubewarden Monitor Violations ==="
kubectl get events -A \
  --field-selector reason=PolicyViolation \
  | grep kubewarden
```

## Step 5: Cut Over

```bash
#!/bin/bash
# cutover-from-kyverno.sh

KYVERNO_POLICY="$1"
KUBEWARDEN_POLICY_NAME="$2"

# Enable Kubewarden policy enforcement
kubectl patch clusteradmissionpolicy "${KUBEWARDEN_POLICY_NAME}" \
  --type=merge \
  -p '{"spec":{"mode":"protect"}}'

echo "Kubewarden policy ${KUBEWARDEN_POLICY_NAME} is now enforcing"

# Disable the Kyverno policy
kubectl patch clusterpolicy "${KYVERNO_POLICY}" \
  --type=merge \
  -p '{"spec":{"validationFailureAction":"Audit"}}'

echo "Kyverno policy ${KYVERNO_POLICY} switched to Audit mode"

# After validation period, delete the Kyverno policy
# kubectl delete clusterpolicy "${KYVERNO_POLICY}"
```

## Step 6: Final Kyverno Removal

After all policies are migrated:

```bash
# Remove all Kyverno policies
kubectl delete clusterpolicies --all
kubectl delete policies -A --all

# Uninstall Kyverno
helm uninstall kyverno -n kyverno
kubectl delete namespace kyverno

echo "Kyverno removed. Migration to Kubewarden complete."
```

## Conclusion

Migrating from Kyverno to Kubewarden is straightforward for validation policies where direct hub equivalents exist, and more involved for custom mutation policies that require Wasm development. The side-by-side approach - running both systems simultaneously with Kubewarden in monitor mode - is the safest migration path. Kyverno's `Generate` rules (for creating Kubernetes resources based on events) have no equivalent in Kubewarden and should be handled separately through other mechanisms such as operators or GitOps controllers.
