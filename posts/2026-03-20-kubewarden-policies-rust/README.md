# How to Write Custom Kubewarden Policies in Rust - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Kubewarden, Rust, Policy as Code, Kubernetes, WebAssembly, Admission Control, SUSE Rancher

Description: Learn how to write custom Kubewarden admission control policies in Rust compiled to WebAssembly, including policy logic, settings validation, and testing with the kwctl tool.

---

Kubewarden policies are WebAssembly modules. Writing them in Rust gives you type safety, excellent performance, and access to the Kubewarden SDK for Kubernetes object inspection.

---

## Prerequisites

```bash
# Install Rust

curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source ~/.cargo/env

# Add the WASM target
rustup target add wasm32-wasi

# Install kwctl (Kubewarden CLI)
curl -Lo kwctl https://github.com/kubewarden/kwctl/releases/latest/download/kwctl-linux-amd64
chmod +x kwctl && sudo mv kwctl /usr/local/bin/
```

---

## Step 1: Create a New Policy Project

```bash
# Use the Kubewarden policy template
cargo generate --git https://github.com/kubewarden/policy-rust-template \
  --name require-resource-limits

cd require-resource-limits
```

The generated project includes:
- `src/lib.rs` - policy logic
- `src/settings.rs` - configurable policy settings
- `metadata.yaml` - policy metadata

---

## Step 2: Write the Policy Logic

```rust
// src/lib.rs
use kubewarden_policy_sdk::prelude::*;
use k8s_openapi::api::core::v1 as apicore;

// Called by the policy SDK for each admission request
#[kubewarden_sdk::policy_entrypoint]
pub fn validate(payload: &[u8]) -> Result<ValidationResponse, ValidationError> {
    // Deserialize the admission request
    let request: ValidationRequest<apicore::Pod> = serde_json::from_slice(payload)?;

    // Get the pod from the request
    let pod = match request.request.object {
        Some(p) => p,
        None => return Ok(ValidationResponse::accept()),
    };

    // Check each container has resource limits
    for container in pod.spec.unwrap_or_default().containers {
        let resources = container.resources.unwrap_or_default();
        let limits = resources.limits.unwrap_or_default();

        // Reject if CPU limit is missing
        if !limits.contains_key("cpu") {
            return Ok(ValidationResponse::reject(
                format!(
                    "Container '{}' must define a CPU limit",
                    container.name
                )
            ));
        }

        // Reject if memory limit is missing
        if !limits.contains_key("memory") {
            return Ok(ValidationResponse::reject(
                format!(
                    "Container '{}' must define a memory limit",
                    container.name
                )
            ));
        }
    }

    Ok(ValidationResponse::accept())
}
```

---

## Step 3: Build the WASM Policy

```bash
# Build the policy as a WASM module
cargo build --target wasm32-wasi --release

# The WASM file is at:
ls target/wasm32-wasi/release/require_resource_limits.wasm
```

---

## Step 4: Test the Policy Locally

```bash
# Create a test request for a pod without limits
cat > test-pod-no-limits.json << EOF
{
  "apiVersion": "admission.k8s.io/v1",
  "kind": "AdmissionReview",
  "request": {
    "uid": "test-123",
    "kind": {"group":"","version":"v1","kind":"Pod"},
    "operation": "CREATE",
    "object": {
      "apiVersion": "v1",
      "kind": "Pod",
      "spec": {
        "containers": [{"name": "app", "image": "nginx"}]
      }
    }
  }
}
EOF

# Run the policy against the test request
kwctl run \
  annotated-policy.wasm \
  --request-path test-pod-no-limits.json

# Expected: policy should REJECT (no limits defined)
```

---

## Step 5: Annotate and Push the Policy

```bash
# Annotate the WASM file with metadata
kwctl annotate \
  target/wasm32-wasi/release/require_resource_limits.wasm \
  --metadata-path metadata.yaml \
  --output annotated-policy.wasm

# Push to an OCI registry
kwctl push \
  annotated-policy.wasm \
  ghcr.io/my-org/require-resource-limits:v0.1.0
```

---

## Best Practices

- Write unit tests in Rust for your policy logic using the Kubewarden test helpers.
- Keep policies focused on one concern - smaller policies are easier to understand and test.
- Use `settings.rs` to make policies configurable rather than hardcoding values.
- Run `kwctl inspect <policy>` to verify the policy metadata before deploying to production.
