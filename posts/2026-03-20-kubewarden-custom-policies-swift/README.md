# How to Write Custom Kubewarden Policies in Swift

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Kubewarden, Kubernetes, Policy, Swift, WebAssembly

Description: Learn how to write custom Kubewarden admission control policies in Swift using Swift's WebAssembly support and the Kubewarden Swift SDK.

## Introduction

Swift's growing WebAssembly support makes it possible to write Kubewarden admission control policies for teams familiar with Apple's ecosystem and Swift programming language. The Swift WebAssembly toolchain can compile Swift code to WASI-compatible WebAssembly, which Kubewarden can then execute as policy.

This guide covers setting up the Swift Wasm toolchain and writing a functional Kubewarden policy in Swift.

## Prerequisites

- Swift 5.9 or later with WebAssembly support
- `carton` build tool (Swift Wasm build tool)
- `kwctl` CLI installed
- Basic Swift programming knowledge

## Setting Up the Swift WebAssembly Environment

```bash
# Install the Swift WebAssembly toolchain
# Download from: https://github.com/swiftwasm/swift/releases

# macOS - install via swiftenv
git clone https://github.com/kylef/swiftenv.git ~/.swiftenv
echo 'export SWIFTENV_ROOT="$HOME/.swiftenv"' >> ~/.bashrc
echo 'export PATH="$SWIFTENV_ROOT/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

# Install the SwiftWasm toolchain
swiftenv install wasm-5.9.0-RELEASE

# Install carton (Swift Wasm build tool)
brew install carton

# Verify installation
swift --version
```

## Creating a New Swift Policy Project

```bash
# Create the policy project
mkdir my-swift-policy && cd my-swift-policy

# Initialize Swift package
swift package init --type executable --name MyKubewardenPolicy

# Update Package.swift for WebAssembly
```

## Configuring Package.swift

```swift
// Package.swift
// swift-tools-version: 5.9
import PackageDescription

let package = Package(
    name: "MyKubewardenPolicy",
    platforms: [
        // WebAssembly target
        .wasi(.v1)
    ],
    dependencies: [
        // Kubewarden Swift SDK
        .package(
            url: "https://github.com/kubewarden/policy-sdk-swift.git",
            from: "0.1.0"
        ),
    ],
    targets: [
        .executableTarget(
            name: "MyKubewardenPolicy",
            dependencies: [
                .product(name: "KubewardenSDK", package: "policy-sdk-swift")
            ],
            path: "Sources"
        ),
    ]
)
```

## Writing the Policy

```swift
// Sources/main.swift
// Kubewarden policy written in Swift
// Policy: Require all pods to have specific annotations

import Foundation
import KubewardenSDK

// Policy settings structure
struct PolicySettings: Codable {
    // Annotations that must be present on all pods
    let requiredAnnotations: [String]

    // Default settings
    init() {
        requiredAnnotations = ["team", "app-version"]
    }
}

// Main entry point for the policy
@_silgen_name("validate")
func validate() {
    do {
        // Read the validation request from the Kubewarden host
        let payload = try KubewardenHost.read()
        let request = try JSONDecoder().decode(ValidationRequest.self, from: payload)

        // Parse settings
        let settings: PolicySettings
        if let settingsData = request.settings {
            settings = (try? JSONDecoder().decode(PolicySettings.self, from: settingsData))
                ?? PolicySettings()
        } else {
            settings = PolicySettings()
        }

        // Extract the Kubernetes object
        guard let objectData = try? JSONEncoder().encode(request.request.object) else {
            KubewardenHost.acceptRequest()
            return
        }

        // Parse the Pod
        guard let pod = try? JSONDecoder().decode(Pod.self, from: objectData) else {
            // Not a Pod, accept
            KubewardenHost.acceptRequest()
            return
        }

        // Check required annotations
        let annotations = pod.metadata?.annotations ?? [:]

        for required in settings.requiredAnnotations {
            if annotations[required] == nil {
                KubewardenHost.rejectRequest(
                    message: "Pod is missing required annotation: '\(required)'",
                    code: 403
                )
                return
            }
        }

        KubewardenHost.acceptRequest()

    } catch {
        KubewardenHost.rejectRequest(
            message: "Policy validation error: \(error)",
            code: 500
        )
    }
}

// Validate policy settings
@_silgen_name("validate_settings")
func validateSettings() {
    do {
        let payload = try KubewardenHost.read()
        let settings = try JSONDecoder().decode(PolicySettings.self, from: payload)

        if settings.requiredAnnotations.isEmpty {
            KubewardenHost.rejectSettings(
                message: "requiredAnnotations cannot be empty"
            )
            return
        }

        KubewardenHost.acceptSettings()

    } catch {
        KubewardenHost.rejectSettings(message: "Invalid settings: \(error)")
    }
}

// Pod data structures (simplified)
struct Pod: Codable {
    let metadata: PodMetadata?
    let spec: PodSpec?
}

struct PodMetadata: Codable {
    let name: String?
    let annotations: [String: String]?
    let labels: [String: String]?
}

struct PodSpec: Codable {
    let containers: [Container]
}

struct Container: Codable {
    let name: String
    let image: String?
}
```

## A More Practical Policy: Validate Security Context

```swift
// Sources/security-context-policy.swift
// Check that containers don't run as root

@_silgen_name("validate")
func validateSecurityContext() {
    guard let payload = try? KubewardenHost.read(),
          let request = try? JSONDecoder().decode(ValidationRequest.self, from: payload) else {
        KubewardenHost.rejectRequest(message: "Cannot parse request", code: 400)
        return
    }

    guard let objectData = try? JSONEncoder().encode(request.request.object),
          let pod = try? JSONDecoder().decode(Pod.self, from: objectData) else {
        KubewardenHost.acceptRequest()
        return
    }

    // Check pod-level security context
    if let podSecCtx = pod.spec?.securityContext {
        if let runAsRoot = podSecCtx.runAsNonRoot, !runAsRoot {
            KubewardenHost.rejectRequest(
                message: "Pod security context explicitly allows running as root",
                code: 403
            )
            return
        }
    }

    // Check container-level security contexts
    for container in pod.spec?.containers ?? [] {
        if let secCtx = container.securityContext {
            // Block explicitly privileged containers
            if let privileged = secCtx.privileged, privileged {
                KubewardenHost.rejectRequest(
                    message: "Container '\(container.name)' is privileged",
                    code: 403
                )
                return
            }

            // Block running as root (UID 0)
            if let runAsUser = secCtx.runAsUser, runAsUser == 0 {
                KubewardenHost.rejectRequest(
                    message: "Container '\(container.name)' runs as root (UID 0)",
                    code: 403
                )
                return
            }
        }
    }

    KubewardenHost.acceptRequest()
}
```

## Building the Policy

```bash
# Build for WebAssembly using carton
carton build --product MyKubewardenPolicy

# Or use swift directly with wasm target
swift build \
  --triple wasm32-unknown-wasi \
  -c release

# Locate the built Wasm file
find .build -name "*.wasm"

# Annotate with Kubewarden metadata
kwctl annotate \
  .build/release/MyKubewardenPolicy.wasm \
  --metadata-path metadata.yml \
  --output annotated-policy.wasm
```

## Policy Metadata

```yaml
# metadata.yml
rules:
  - apiGroups: [""]
    apiVersions: ["v1"]
    resources: ["pods"]
    operations:
      - CREATE
      - UPDATE
mutating: false
contextAware: false
executionMode: kubewarden-wapc
annotations:
  io.kubewarden.policy.title: require-pod-annotations
  io.kubewarden.policy.description: Require specific annotations on all pods
  io.kubewarden.policy.author: Platform Team
  io.kubewarden.policy.severity: low
```

## Testing the Policy

```bash
# Test with kwctl
kwctl run \
  annotated-policy.wasm \
  --request-path tests/missing-annotations.json \
  --settings-json '{"requiredAnnotations": ["team", "app-version"]}'
```

## Publishing the Swift Policy

```bash
# Push to OCI registry
kwctl push \
  annotated-policy.wasm \
  registry.example.com/kubewarden/require-pod-annotations:v0.1.0
```

## Conclusion

Writing Kubewarden policies in Swift enables Apple platform developers and Swift enthusiasts to contribute to Kubernetes security governance using their preferred language. While the Swift WebAssembly ecosystem is still maturing compared to Rust and Go, the combination of Swift's expressive syntax, type safety, and growing Wasm support makes it a viable option for teams invested in the Swift ecosystem. As Swift's WASI support continues to improve, writing Kubernetes admission policies in Swift will become increasingly straightforward.
