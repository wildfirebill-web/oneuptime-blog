# How to Migrate from OPA Gatekeeper to Kubewarden

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Kubewarden, Kubernetes, OPA, Gatekeeper, Migration

Description: A step-by-step guide to migrating your Kubernetes admission control policies from OPA Gatekeeper to Kubewarden, covering policy translation and side-by-side migration strategies.

## Introduction

OPA Gatekeeper and Kubewarden both provide Kubernetes admission control, but they take different approaches: Gatekeeper uses Rego policies and the Open Policy Agent engine, while Kubewarden uses WebAssembly policies that can be written in any language. Organizations migrating to Kubewarden benefit from language flexibility, better performance through Wasm execution, and a simpler configuration model.

This guide covers migrating from Gatekeeper to Kubewarden with minimal disruption.

## Prerequisites

- Existing OPA Gatekeeper installation
- Kubewarden installed (can run alongside Gatekeeper initially)
- `kubectl` with cluster-admin access
- Inventory of existing Gatekeeper policies

## Understanding the Differences

| Feature | OPA Gatekeeper | Kubewarden |
|---------|----------------|------------|
| Policy language | Rego | Any language (Rust, Go, AssemblyScript) |
| Policy format | ConfigMap/CRD | WebAssembly modules (OCI) |
| Mutation support | Limited | Full |
| Testing | OPA CLI | kwctl |
| Context-aware | Via sync config | Native host calls |
| Performance | Rego interpretation | Wasm near-native |

## Step 1: Inventory Your Gatekeeper Policies

```bash
# List all Gatekeeper constraint templates
kubectl get constrainttemplates

# List all Gatekeeper constraints (instances)
kubectl get constraints -A 2>/dev/null || \
  kubectl get $(kubectl get constrainttemplates \
    -o jsonpath='{.items[*].metadata.name}' | tr ' ' ',') -A 2>/dev/null

# Export all constraint templates
kubectl get constrainttemplates -o yaml > gatekeeper-templates.yaml

# Export all constraints
for template in $(kubectl get constrainttemplates \
  -o jsonpath='{.items[*].metadata.name}'); do
  kubectl get "$template" -A -o yaml >> gatekeeper-constraints.yaml
  echo "---" >> gatekeeper-constraints.yaml
done
```

## Step 2: Map Gatekeeper Policies to Kubewarden Equivalents

Many common Gatekeeper policies have direct Kubewarden equivalents:

### Privileged Containers

**Gatekeeper version:**
```yaml
# K8sPSPPrivilegedContainer (Gatekeeper)
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sPSPPrivilegedContainer
metadata:
  name: psp-privileged-container
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
```

**Kubewarden equivalent:**
```yaml
# pod-privileged (Kubewarden)
apiVersion: policies.kubewarden.io/v1
kind: ClusterAdmissionPolicy
metadata:
  name: no-privileged-containers
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

### Allowed Registries

**Gatekeeper version:**
```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sAllowedRepos
metadata:
  name: repo-is-allowed
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
  parameters:
    repos:
      - "registry.internal.example.com"
      - "gcr.io/my-project"
```

**Kubewarden equivalent:**
```yaml
apiVersion: policies.kubewarden.io/v1
kind: ClusterAdmissionPolicy
metadata:
  name: allowed-registries
spec:
  module: registry://ghcr.io/kubewarden/policies/allowed-image-repositories:v0.1.0
  settings:
    allowedRegistries:
      - "registry.internal.example.com"
      - "gcr.io/my-project"
  rules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      resources: ["pods"]
      operations: ["CREATE", "UPDATE"]
  mutating: false
  mode: protect
```

### Required Labels

**Gatekeeper version:**
```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: require-team-label
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Namespace"]
  parameters:
    labels:
      - key: "team"
```

**Kubewarden equivalent:**
```yaml
apiVersion: policies.kubewarden.io/v1
kind: ClusterAdmissionPolicy
metadata:
  name: require-team-label
spec:
  module: registry://ghcr.io/kubewarden/policies/k8s-objects:v1.3.0
  settings:
    mandatory_labels:
      - team
  rules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      resources: ["namespaces"]
      operations: ["CREATE", "UPDATE"]
  mutating: false
  mode: protect
```

## Step 3: Implement Side-by-Side Migration

Run Kubewarden in monitor mode alongside Gatekeeper:

```bash
# Phase 1: Deploy Kubewarden policies in MONITOR mode
# while Gatekeeper remains in ENFORCE mode

for policy_file in kubewarden-policies/*.yaml; do
  # Ensure all policies are in monitor mode
  sed 's/mode: protect/mode: monitor/' "$policy_file" | \
    kubectl apply -f -
done

# Phase 2: Compare violations between the two systems
echo "=== Gatekeeper violations ==="
kubectl get events -A \
  --field-selector reason=FailedCreate \
  | grep "admission webhook" \
  | grep gatekeeper

echo "=== Kubewarden monitor violations ==="
kubectl get events -A \
  --field-selector reason=PolicyViolation
```

## Step 4: Gradual Transition

```bash
#!/bin/bash
# transition-policy.sh - Transition one policy at a time

POLICY_NAME="$1"
GATEKEEPER_CONSTRAINT="$2"

echo "Transitioning: ${GATEKEEPER_CONSTRAINT} -> ${POLICY_NAME}"

# Step 1: Enable Kubewarden policy in protect mode
kubectl patch clusteradmissionpolicy "${POLICY_NAME}" \
  --type=merge \
  -p '{"spec":{"mode":"protect"}}'

echo "Kubewarden policy ${POLICY_NAME} now in PROTECT mode"

# Step 2: Test that the Kubewarden policy works
# (run your workload tests here)

# Step 3: Disable the Gatekeeper constraint
kubectl annotate constraint "${GATEKEEPER_CONSTRAINT}" \
  "gatekeeper.sh/disable=true" \
  --overwrite

echo "Gatekeeper constraint ${GATEKEEPER_CONSTRAINT} disabled"
echo "Transition complete. Monitor for 24h before removing Gatekeeper constraint"
```

## Step 5: Remove Gatekeeper

After all policies are migrated and validated:

```bash
# Remove all Gatekeeper constraints
kubectl delete constraints -A --all 2>/dev/null || true

# Remove all constraint templates
kubectl delete constrainttemplates --all

# Uninstall Gatekeeper
helm uninstall gatekeeper -n gatekeeper-system

# Remove Gatekeeper namespace
kubectl delete namespace gatekeeper-system

echo "Gatekeeper removed. Kubewarden is now your sole admission controller."
```

## Handling Custom Rego Policies

For custom Rego policies without Kubewarden equivalents, you need to rewrite them in Rust, Go, or AssemblyScript. Use this approach:

```bash
# Export the Rego policy for reference
kubectl get constrainttemplate my-custom-policy \
  -o jsonpath='{.spec.targets[0].rego}'

# Create a new Kubewarden policy project
cargo generate \
  --git https://github.com/kubewarden/rust-policy-template \
  --name my-custom-policy

# Translate the Rego logic to Rust/Go
# (Refer to Kubewarden documentation for translation patterns)
```

## Validating the Migration

```bash
# Run your full test suite against the cluster
# with only Kubewarden active

# Check for any unexpected denials in the last 24h
kubectl get events -A \
  --field-selector reason=PolicyViolation \
  --sort-by='.lastTimestamp' \
  | tail -50

# Verify all workloads are running
kubectl get pods -A | grep -v Running | grep -v Completed
```

## Conclusion

Migrating from OPA Gatekeeper to Kubewarden requires careful planning but the benefits — language flexibility, WebAssembly performance, and simpler configuration — are significant. The side-by-side migration approach minimizes risk by running both systems simultaneously in monitor vs. enforce mode, giving you confidence in the Kubewarden policies before removing Gatekeeper. For common policies, the Kubewarden Policy Hub provides ready-to-use replacements, while custom Rego policies require a rewrite but gain the benefits of type safety and better tooling.
