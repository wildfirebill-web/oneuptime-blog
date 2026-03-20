# Kubewarden vs OPA Gatekeeper: Policy Engine Comparison

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Kubewarden, Opa-gatekeeper, Policy-engine, Kubernetes, Comparison

Description: A comprehensive comparison of Kubewarden and OPA Gatekeeper for Kubernetes policy enforcement, covering policy authoring, performance, and operational experience.

## Overview

Kubernetes policy engines enforce governance, compliance, and security constraints on cluster resources. Kubewarden and OPA Gatekeeper are two leading policy engines with different approaches. OPA Gatekeeper uses the Rego policy language, while Kubewarden uses WebAssembly (Wasm) modules written in any supported language. This guide compares them to help you choose the right policy engine.

## What Is OPA Gatekeeper?

OPA Gatekeeper is a CNCF-graduated admission controller that uses Open Policy Agent (OPA) and the Rego policy language to enforce policies. It integrates with Kubernetes via webhooks and provides ConstraintTemplates (policy definitions) and Constraints (policy instances). It is widely adopted and has a large ecosystem of pre-built policies.

## What Is Kubewarden?

Kubewarden is a CNCF Sandbox policy engine from SUSE Rancher that uses WebAssembly (Wasm) modules for policies. Policies can be written in Go, Rust, Python, Swift, AssemblyScript, or any language that compiles to Wasm. Policies are distributed via OCI registries.

## Feature Comparison

| Feature | Kubewarden | OPA Gatekeeper |
|---|---|---|
| Policy Language | Any (Wasm-compiled) | Rego only |
| Policy Distribution | OCI registry | Inline (YAML) |
| Policy Testing | Yes (kwctl tool) | Yes (opa test) |
| Audit Mode | Yes | Yes |
| Mutation Policies | Yes | Yes (experimental) |
| Context-Aware Policies | Yes | Yes |
| Policy Hub | Yes (ArtifactHub) | Yes (gatekeeper-library) |
| CNCF Status | Sandbox | Graduated |
| Rancher Integration | Native UI | Via kubectl |
| Performance | High (Wasm) | Good (Rego) |
| Learning Curve | Medium (learn Wasm toolchain) | Medium (learn Rego) |
| Community Size | Growing | Large |

## Policy Definition Comparison

### OPA Gatekeeper Policy

Gatekeeper uses ConstraintTemplates (written in Rego) and Constraints (instances):

```yaml
# ConstraintTemplate: Require labels

apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels
      validation:
        openAPIV3Schema:
          type: object
          properties:
            labels:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredlabels
        violation[{"msg": msg, "details": {"missing_labels": missing}}] {
          provided := {label | input.review.object.metadata.labels[label]}
          required := {label | label := input.parameters.labels[_]}
          missing := required - provided
          count(missing) > 0
          msg := sprintf("Missing required labels: %v", [missing])
        }
---
# Constraint: Enforce the policy
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
    labels: ["team", "cost-center"]
```

### Kubewarden Policy

Kubewarden policies are compiled Wasm binaries deployed as ClusterAdmissionPolicy or AdmissionPolicy:

```yaml
# Kubewarden ClusterAdmissionPolicy using a pre-built Wasm policy
apiVersion: policies.kubewarden.io/v1
kind: ClusterAdmissionPolicy
metadata:
  name: require-labels
spec:
  module: registry://ghcr.io/kubewarden/policies/require-labels:v0.2.0
  rules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      resources: ["namespaces"]
      operations: ["CREATE", "UPDATE"]
  mutating: false
  settings:
    mandatory_labels:
      - team
      - cost-center
```

## Writing Custom Policies

### Kubewarden Policy in Go

```go
// custom-policy/main.go
package main

import (
    "github.com/kubewarden/k8s-objects/api/core/v1"
    "github.com/kubewarden/policy-sdk-go"
)

func validate(payload []byte) ([]byte, error) {
    // Parse the admission request
    admissionRequest, err := kubewarden.UnmarshalAdmissionRequest(payload)
    if err != nil {
        return kubewarden.RejectRequest(
            kubewarden.Message(err.Error()), nil)
    }
    // Custom validation logic here
    return kubewarden.AcceptRequest()
}

func main() {
    kubewarden.WasiEntryPoint(validate)
}
```

### Testing Kubewarden Policies

```bash
# Use kwctl to test policies locally
kwctl run registry://ghcr.io/kubewarden/policies/require-labels:v0.2.0 \
  --settings-json '{"mandatory_labels": ["team"]}' \
  --request-path test-request.json

# Test with a failing request
kwctl run annotated-policy.wasm \
  --request-path test/namespace-no-label.json \
  # Expected: REJECTED
```

## Audit Scanning

Both engines support audit mode to identify existing violations:

```bash
# Gatekeeper audit results
kubectl get constraints -A
kubectl describe k8srequiredlabels require-team-label
# Shows violations in .status.violations

# Kubewarden audit scan
kubectl get clusteradmissionpolicies
kubectl describe clusteradmissionpolicy require-labels
# Shows audit results in status
```

## Performance

Kubewarden's Wasm-based policies execute in an isolated sandbox. WebAssembly runtime overhead is minimal, and complex policies written in compiled languages (Rust, Go) can be very fast.

OPA Gatekeeper uses Rego, an interpreted language designed for policy evaluation. For most policies, performance is adequate. Very complex Rego policies with large data sets may be slower than equivalent Wasm policies.

## When to Choose Kubewarden

- Your team is comfortable with Go, Rust, or other compiled languages
- You want to write and publish policies using standard OCI tooling
- Native Rancher UI integration is important
- You want maximum flexibility in policy implementation language

## When to Choose OPA Gatekeeper

- CNCF Graduated status and ecosystem maturity are priorities
- You are already familiar with Rego
- The gatekeeper-library provides ready-made policies you need
- You want a large community and existing documentation

## Conclusion

Both Kubewarden and OPA Gatekeeper are capable Kubernetes policy engines. Kubewarden's strength is its use of WebAssembly, allowing policies in any language with excellent performance and easy distribution via OCI registries. OPA Gatekeeper's strength is its maturity, large community, and the extensive Rego policy ecosystem. Teams using Rancher will benefit from Kubewarden's native integration, while teams with existing Rego expertise should consider sticking with OPA Gatekeeper.
