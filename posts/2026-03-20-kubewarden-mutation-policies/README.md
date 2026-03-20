# How to Configure Kubewarden Mutation Policies

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Kubewarden, Kubernetes, Policy, Mutation, Security

Description: Learn how to write and deploy Kubewarden mutation policies that automatically modify Kubernetes resources to enforce security defaults and organizational standards.

## Introduction

Kubewarden mutation policies automatically modify Kubernetes resources as they are created or updated, injecting security settings, adding labels, or adjusting configurations without requiring users to specify every detail. Unlike validation policies that simply allow or deny requests, mutation policies transform resources to comply with organizational standards automatically.

This guide covers creating, deploying, and testing Kubewarden mutation policies.

## Prerequisites

- Kubewarden installed on your cluster
- `kubectl` access with cluster-admin permissions
- Basic understanding of Kubernetes JSON patches

## Understanding Mutation Policies

Mutation policies return a JSON patch that modifies the resource before it is persisted. The key setting is `mutating: true` in the policy definition.

Common mutation use cases:
- Inject security context defaults
- Add required labels and annotations
- Set default resource requests and limits
- Add sidecar containers
- Set image pull policies

## Deploying a Hub Mutation Policy

### Auto-Inject Security Context

The `pod-runtime-class` policy mutates pods to add security context defaults:

```yaml
# mutation-security-context.yaml

apiVersion: policies.kubewarden.io/v1
kind: ClusterAdmissionPolicy
metadata:
  name: mutate-add-security-context
spec:
  module: registry://ghcr.io/kubewarden/policies/pod-privileged:v0.2.0

  # CRITICAL: Set mutating to true for mutation policies
  mutating: true

  rules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      resources: ["pods"]
      operations:
        - CREATE
  mode: protect
```

### Auto-Add Labels Policy

```yaml
# mutation-add-labels.yaml
apiVersion: policies.kubewarden.io/v1
kind: ClusterAdmissionPolicy
metadata:
  name: mutate-add-labels
spec:
  module: registry://ghcr.io/kubewarden/policies/safe-labels:v0.1.0

  mutating: true

  settings:
    mandatory_labels:
      "managed-by": "kubewarden"

  rules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      resources: ["pods"]
      operations: ["CREATE"]
  mode: protect
```

## Writing a Custom Mutation Policy in Rust

```rust
// src/lib.rs - Mutation policy that adds security context defaults

use anyhow::{anyhow, Result};
use guest::prelude::*;
use kubewarden_policy_sdk::wapc_guest as guest;

use k8s_openapi::api::core::v1 as apicore;

extern crate kubewarden_policy_sdk as kubewarden;
use kubewarden::{
    mutation_response,
    protocol_version_guest,
    request::ValidationRequest,
    validate_settings,
};

mod settings;
use settings::Settings;

#[no_mangle]
pub extern "C" fn wapc_init() {
    register_function("validate", validate);
    register_function("validate_settings", validate_settings::<Settings>);
    register_function("protocol_version", protocol_version_guest);
}

fn validate(payload: &[u8]) -> CallResult {
    let validation_request: ValidationRequest<Settings> =
        serde_json::from_slice(payload).map_err(|e| {
            anyhow!("Cannot parse payload: {}", e)
        })?;

    // Parse the Pod
    let mut pod: apicore::Pod =
        serde_json::from_value(validation_request.request.object.clone())
        .map_err(|_| anyhow!("Cannot parse Pod"))?;

    // Apply mutations to add security defaults
    let mutated = apply_security_defaults(&mut pod);

    if mutated {
        // Return a mutation response with the modified pod
        let mutated_pod_json = serde_json::to_value(&pod)
            .map_err(|e| anyhow!("Cannot serialize mutated pod: {}", e))?;

        return kubewarden::mutate_request(mutated_pod_json);
    }

    // No mutations needed, accept as-is
    kubewarden::accept_request()
}

fn apply_security_defaults(pod: &mut apicore::Pod) -> bool {
    let mut mutated = false;

    // Ensure pod spec exists
    let spec = match pod.spec.as_mut() {
        Some(s) => s,
        None => return false,
    };

    // Apply security context defaults to each container
    for container in &mut spec.containers {
        let sec_ctx = container.security_context.get_or_insert_with(Default::default);

        // Set allowPrivilegeEscalation to false if not specified
        if sec_ctx.allow_privilege_escalation.is_none() {
            sec_ctx.allow_privilege_escalation = Some(false);
            mutated = true;
        }

        // Set readOnlyRootFilesystem to true if not specified
        if sec_ctx.read_only_root_filesystem.is_none() {
            sec_ctx.read_only_root_filesystem = Some(true);
            mutated = true;
        }

        // Set runAsNonRoot to true if not specified
        if sec_ctx.run_as_non_root.is_none() {
            sec_ctx.run_as_non_root = Some(true);
            mutated = true;
        }
    }

    mutated
}
```

## Testing Mutation Policies

```bash
# Create a test pod without security context
cat > test-mutation.json <<'EOF'
{
  "request": {
    "uid": "mutation-test-001",
    "kind": {"group": "", "version": "v1", "kind": "Pod"},
    "operation": "CREATE",
    "object": {
      "apiVersion": "v1",
      "kind": "Pod",
      "metadata": {"name": "test-pod"},
      "spec": {
        "containers": [
          {"name": "app", "image": "nginx:1.25.0"}
        ]
      }
    }
  }
}
EOF

# Run the mutation policy
kwctl run ./build/mutation-policy.wasm --request-path test-mutation.json

# The response should contain a JSON patch
# showing the added security context fields
```

## Viewing Mutation Results

```bash
# Create a pod and check if mutations were applied
kubectl run test-pod --image=nginx:1.25.0 --dry-run=server -o json \
  | jq '.spec.containers[0].securityContext'

# Expected output (if mutation policy is active):
# {
#   "allowPrivilegeEscalation": false,
#   "readOnlyRootFilesystem": true,
#   "runAsNonRoot": true
# }
```

## Mutation vs Validation Policy Order

Kubewarden processes mutation policies before validation policies. This means:
1. Mutation policies modify the resource
2. Validation policies evaluate the (possibly mutated) resource

This allows you to:
- Mutate resources to comply with standards
- Validate that standards are met (after mutation)

```yaml
# First: mutate to add security defaults
apiVersion: policies.kubewarden.io/v1
kind: ClusterAdmissionPolicy
metadata:
  name: mutate-add-security-defaults
spec:
  module: registry://example.com/policies/add-security-context:v0.1.0
  mutating: true
  rules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      resources: ["pods"]
      operations: ["CREATE"]
  mode: protect
---
# Then: validate the security context is present
apiVersion: policies.kubewarden.io/v1
kind: ClusterAdmissionPolicy
metadata:
  name: validate-security-context
spec:
  module: registry://example.com/policies/require-security-context:v0.1.0
  mutating: false
  rules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      resources: ["pods"]
      operations: ["CREATE", "UPDATE"]
  mode: protect
```

## Conclusion

Kubewarden mutation policies enable you to automatically enforce organizational standards without requiring every developer to know every security setting. By combining mutation policies that add defaults with validation policies that verify compliance, you create a system where the platform makes it easy to do the right thing while still catching cases where explicit insecure configurations are requested. This combination is more user-friendly than pure validation because it reduces friction for compliant workloads while maintaining security guarantees.
