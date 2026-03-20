# How to Write Custom Kubewarden Policies in Swift - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Kubewarden, Swift, Policy as Code, Kubernetes, WebAssembly, Admission Control, SUSE Rancher

Description: Learn how to write custom Kubewarden admission policies in Swift compiled to WebAssembly using SwiftWasm, enabling Swift developers to contribute to Kubernetes policy authoring.

---

Swift can compile to WebAssembly via the SwiftWasm toolchain, making it possible to write Kubewarden policies using Swift's expressive syntax and strong type system.

---

## Prerequisites

```bash
# Install SwiftWasm toolchain

SWIFT_WASM_VERSION="wasm-5.9.1-RELEASE"
curl -Lo swift-wasm.tar.gz \
  "https://github.com/swiftwasm/swift/releases/download/${SWIFT_WASM_VERSION}/swift-wasm-5.9.1-RELEASE-ubuntu22.04_aarch64.tar.gz"
tar xf swift-wasm.tar.gz
export PATH=$PWD/usr/bin:$PATH

# Verify SwiftWasm is available
swift --version | grep wasm

# Install kwctl for testing
curl -Lo kwctl https://github.com/kubewarden/kwctl/releases/latest/download/kwctl-linux-amd64
chmod +x kwctl && sudo mv kwctl /usr/local/bin/
```

---

## Step 1: Set Up the Policy Project

```bash
mkdir label-enforcement-policy
cd label-enforcement-policy

# Create Swift package
swift package init --type executable

# Add required dependencies to Package.swift
cat > Package.swift << 'EOF'
// swift-tools-version: 5.9
import PackageDescription

let package = Package(
    name: "LabelEnforcementPolicy",
    dependencies: [
        .package(
            url: "https://github.com/kubewarden/policy-sdk-swift.git",
            from: "0.1.0"
        )
    ],
    targets: [
        .executableTarget(
            name: "LabelEnforcementPolicy",
            dependencies: [
                .product(name: "KubewardenPolicySDK", package: "policy-sdk-swift")
            ]
        )
    ]
)
EOF
```

---

## Step 2: Write the Policy Logic

```swift
// Sources/LabelEnforcementPolicy/main.swift
import KubewardenPolicySDK
import Foundation

// Policy validates that all pods have required labels
func validate(request: ValidationRequest) throws -> ValidationResponse {
    guard let pod = request.object as? Pod else {
        return ValidationResponse.accept()
    }

    // Required labels for all pods
    let requiredLabels = ["app", "version", "team"]

    let labels = pod.metadata?.labels ?? [:]
    var missingLabels: [String] = []

    for label in requiredLabels {
        if labels[label] == nil {
            missingLabels.append(label)
        }
    }

    if !missingLabels.isEmpty {
        return ValidationResponse.reject(
            message: "Pod is missing required labels: \(missingLabels.joined(separator: ", "))"
        )
    }

    return ValidationResponse.accept()
}

// Entry point for the WASM module
KubewardenPolicySDK.run(validate: validate)
```

---

## Step 3: Build the WASM Policy

```bash
# Compile Swift to WASM
swift build \
  --swift-sdk wasm32-unknown-wasi \
  -c release

# Locate the WASM binary
ls .build/wasm32-unknown-wasi/release/LabelEnforcementPolicy.wasm
```

---

## Step 4: Test the Policy

```bash
# Test: pod without required labels (should be rejected)
cat > test-no-labels.json << EOF
{
  "request": {
    "operation": "CREATE",
    "object": {
      "metadata": {"name": "my-pod"},
      "spec": {
        "containers": [{"name": "app", "image": "nginx:1.24"}]
      }
    }
  }
}
EOF

kwctl run \
  .build/wasm32-unknown-wasi/release/LabelEnforcementPolicy.wasm \
  --request-path test-no-labels.json
# Expected: rejected - missing labels

# Test: pod with all labels (should pass)
cat > test-with-labels.json << EOF
{
  "request": {
    "operation": "CREATE",
    "object": {
      "metadata": {
        "name": "my-pod",
        "labels": {"app": "my-app", "version": "v1.0", "team": "platform"}
      },
      "spec": {
        "containers": [{"name": "app", "image": "nginx:1.24"}]
      }
    }
  }
}
EOF

kwctl run \
  .build/wasm32-unknown-wasi/release/LabelEnforcementPolicy.wasm \
  --request-path test-with-labels.json
# Expected: accepted
```

---

## Step 5: Package and Deploy

```bash
# Annotate the policy
kwctl annotate \
  .build/wasm32-unknown-wasi/release/LabelEnforcementPolicy.wasm \
  --metadata-path metadata.yaml \
  --output annotated-policy.wasm

# Push to registry
kwctl push annotated-policy.wasm \
  ghcr.io/my-org/label-enforcement:v0.1.0
```

---

## Best Practices

- Swift's WASM support is newer than Rust or Go - use it for teams already invested in Swift.
- For mission-critical policies, prefer Rust or Go which have more mature Kubewarden SDK support.
- Always write unit tests in pure Swift (without WASM) before compiling - Swift tests run faster than WASM-based tests.
