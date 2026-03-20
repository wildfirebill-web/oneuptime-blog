# Kubewarden vs Kyverno: Policy Engine Comparison

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: kubewarden, kyverno, policy-engine, kubernetes, comparison

Description: A detailed comparison of Kubewarden and Kyverno for Kubernetes policy management, covering policy authoring, mutation, validation, and ease of use.

## Overview

Kyverno and Kubewarden are modern Kubernetes policy engines that take different approaches to policy authoring. Kyverno uses YAML-native policies that are easy to read and write without learning a new language. Kubewarden uses WebAssembly modules that can be written in any compiled language. This comparison helps teams choose the right policy engine for their Kubernetes governance needs.

## What Is Kyverno?

Kyverno is a CNCF-graduated Kubernetes-native policy engine that uses YAML policies without requiring a separate policy language. It supports validation, mutation, generation, and cleanup of Kubernetes resources. Its policies are easy to read and can be applied directly with kubectl.

## What Is Kubewarden?

Kubewarden is a CNCF Sandbox policy engine from SUSE Rancher that uses WebAssembly modules for policy enforcement. It provides flexibility in policy authoring language and distributes policies via OCI registries.

## Feature Comparison

| Feature | Kubewarden | Kyverno |
|---|---|---|
| Policy Language | Any (Wasm) | YAML-native |
| Validation Policies | Yes | Yes |
| Mutation Policies | Yes | Yes |
| Generate Policies | No | Yes |
| Cleanup Policies | No | Yes |
| CEL Support | No | Yes (v1.10+) |
| Context-Aware | Yes | Yes |
| Policy Testing | kwctl | kyverno test |
| Policy Library | ArtifactHub | Kyverno policies (GitHub) |
| CNCF Status | Sandbox | Graduated |
| Rancher Integration | Native | Via kubectl |
| Learning Curve | Medium (Wasm toolchain) | Low (YAML) |
| Community | Growing | Large |
| Background Scans | Yes | Yes |

## Policy Examples

### Kyverno Validation Policy

```yaml
# Kyverno: Require resource limits on all containers
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-limits
spec:
  validationFailureAction: enforce
  rules:
    - name: check-container-limits
      match:
        any:
          - resources:
              kinds:
                - Pod
      validate:
        message: "Resource limits are required for all containers."
        pattern:
          spec:
            containers:
              - (name): "?*"
                resources:
                  limits:
                    memory: "?*"
                    cpu: "?*"
```

### Kubewarden Equivalent (using pre-built policy)

```yaml
# Kubewarden: Require resource limits using ArtifactHub policy
apiVersion: policies.kubewarden.io/v1
kind: ClusterAdmissionPolicy
metadata:
  name: require-resource-limits
spec:
  module: registry://ghcr.io/kubewarden/policies/resource-limits:v0.1.0
  rules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      resources: ["pods"]
      operations: ["CREATE", "UPDATE"]
  mutating: false
  settings:
    cpu_limit_required: true
    memory_limit_required: true
```

## Mutation Policies

### Kyverno Mutation

```yaml
# Kyverno: Auto-add resource limits if missing
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: add-default-limits
spec:
  rules:
    - name: add-memory-limit
      match:
        any:
          - resources:
              kinds:
                - Pod
      mutate:
        patchStrategicMerge:
          spec:
            containers:
              - (name): "?*"
                resources:
                  limits:
                    +(memory): "256Mi"
                    +(cpu): "500m"
```

### Kubewarden Mutation

```yaml
# Kubewarden mutation policy
apiVersion: policies.kubewarden.io/v1
kind: ClusterAdmissionPolicy
metadata:
  name: add-default-limits
spec:
  module: registry://ghcr.io/kubewarden/policies/add-default-limits:v0.1.0
  rules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      resources: ["pods"]
      operations: ["CREATE"]
  mutating: true
  settings:
    default_memory_limit: "256Mi"
    default_cpu_limit: "500m"
```

## Resource Generation (Kyverno Exclusive)

Kyverno can generate new Kubernetes resources in response to events. This is a unique capability not available in Kubewarden:

```yaml
# Kyverno: Auto-create NetworkPolicy when a Namespace is created
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: generate-networkpolicy
spec:
  rules:
    - name: create-default-deny
      match:
        any:
          - resources:
              kinds:
                - Namespace
      generate:
        apiVersion: networking.k8s.io/v1
        kind: NetworkPolicy
        name: default-deny-all
        namespace: "{{request.object.metadata.name}}"
        data:
          spec:
            podSelector: {}
            policyTypes:
              - Ingress
              - Egress
```

## Policy Testing

### Kyverno Test

```bash
# Test Kyverno policies locally
kyverno test .

# Test with a specific resource
kyverno apply require-limits.yaml \
  --resource test-pod.yaml

# Output shows PASS/FAIL for each rule
```

### Kubewarden kwctl

```bash
# Test Kubewarden policies
kwctl run registry://ghcr.io/kubewarden/policies/require-labels:v0.2.0 \
  --settings-json '{"mandatory_labels": ["app"]}' \
  --request-path test-pod.json

# Scaffold a new policy
kwctl scaffold manifest \
  --type ClusterAdmissionPolicy \
  registry://ghcr.io/kubewarden/policies/require-labels:v0.2.0
```

## When to Choose Kyverno

- Your team wants YAML-native policies without learning a new language
- Policy generation (creating new resources) is needed
- CEL expressions for policy logic are preferred
- You want a CNCF-graduated tool with a large community
- Cleanup policies for stale resources are needed

## When to Choose Kubewarden

- Your team writes policies in Go, Rust, or another compiled language
- Policy distribution via OCI registries is preferred
- Native Rancher UI integration is important
- You want maximum performance from compiled Wasm modules

## Conclusion

Kyverno and Kubewarden both provide excellent Kubernetes policy enforcement. Kyverno's key advantage is its low barrier to entry — YAML-native policies require no additional toolchain. Kubewarden's key advantage is flexibility — any language that compiles to Wasm can be used to write policies, which is powerful for complex validation logic. Teams new to policy management should start with Kyverno for its simplicity. Teams with experienced engineers who want language flexibility should consider Kubewarden, especially in Rancher environments.
