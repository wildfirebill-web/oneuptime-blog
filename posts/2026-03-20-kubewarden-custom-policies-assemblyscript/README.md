# How to Write Custom Kubewarden Policies in AssemblyScript

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Kubewarden, Kubernetes, Policy, AssemblyScript, WebAssembly

Description: Learn how to write custom Kubewarden admission control policies using AssemblyScript, a TypeScript-like language that compiles directly to WebAssembly.

## Introduction

AssemblyScript is a variant of TypeScript that compiles directly to WebAssembly. For teams familiar with JavaScript or TypeScript, AssemblyScript offers a familiar syntax for writing Kubewarden policies without needing to learn Rust or Go. The Kubewarden AssemblyScript SDK provides the necessary host functions and types for building admission policies.

## Prerequisites

- Node.js 18 or later
- npm or yarn package manager
- `kwctl` CLI installed
- TypeScript/JavaScript programming knowledge

## Setting Up the Development Environment

```bash
# Install Node.js (if not installed)

# macOS
brew install node

# Verify versions
node --version
npm --version

# Install the AssemblyScript compiler globally
npm install -g assemblyscript

# Install kwctl
curl -LO https://github.com/kubewarden/kwctl/releases/latest/download/kwctl-linux-amd64
chmod +x kwctl-linux-amd64
sudo mv kwctl-linux-amd64 /usr/local/bin/kwctl
```

## Creating a New Policy Project

```bash
# Create project directory
mkdir my-as-policy && cd my-as-policy

# Initialize npm project
npm init -y

# Install Kubewarden AssemblyScript SDK
npm install --save @kubewarden/policy-sdk

# Install AssemblyScript as dev dependency
npm install --save-dev assemblyscript

# Initialize AssemblyScript project
npx asinit . --yes
```

## Project Structure

```text
my-as-policy/
├── package.json
├── asconfig.json           # AssemblyScript config
├── metadata.yml
├── assembly/
│   ├── index.ts            # Main policy logic
│   └── settings.ts         # Policy settings
├── tests/
│   └── data/
│       ├── valid-pod.json
│       └── invalid-pod.json
└── Makefile
```

## Configuring AssemblyScript

```json
// asconfig.json
{
  "targets": {
    "release": {
      "binaryFile": "build/policy.wasm",
      "sourceMap": false,
      "optimizeLevel": 3,
      "noAssert": true
    }
  },
  "options": {
    "exportRuntime": false,
    "runtime": "stub"
  }
}
```

## Writing the Policy

### Main Policy File

```typescript
// assembly/index.ts
// Policy to validate that all containers have resource limits set

import { JSON } from "assemblyscript-json/assembly";
import {
  validate_settings,
  wapc_init,
  ValidationRequest,
  ValidationResponse,
} from "@kubewarden/policy-sdk/assembly";

// Policy settings definition
class Settings {
  // Whether CPU limits are required
  requireCPULimit: bool = true;
  // Whether memory limits are required
  requireMemoryLimit: bool = true;
}

// Entry point for the Kubewarden host
export function validate(payload: ArrayBuffer): ArrayBuffer {
  // Parse the incoming request
  const request = ValidationRequest.fromBuffer(payload);

  if (request == null) {
    return ValidationResponse.reject(
      "Failed to parse validation request",
      400
    ).toBuffer();
  }

  // Parse settings
  const settings = parseSettings(request.settings);

  // Get the Pod object from the request
  const pod = JSON.parse(request.object) as JSON.Obj;
  if (pod == null) {
    // Accept if we can't parse (not a Pod)
    return ValidationResponse.accept().toBuffer();
  }

  // Validate each container's resource configuration
  const spec = pod.getObj("spec");
  if (spec == null) {
    return ValidationResponse.accept().toBuffer();
  }

  const containers = spec.getArr("containers");
  if (containers == null) {
    return ValidationResponse.accept().toBuffer();
  }

  // Check each container
  for (let i = 0; i < containers.valueOf().length; i++) {
    const container = containers.valueOf()[i] as JSON.Obj;
    const name = container.getString("name");
    const containerName = name != null ? name.valueOf() : "unknown";

    const resources = container.getObj("resources");
    if (resources == null) {
      return ValidationResponse.reject(
        `Container '${containerName}' has no resources section defined`,
        403
      ).toBuffer();
    }

    const limits = resources.getObj("limits");

    if (settings.requireCPULimit) {
      if (limits == null || limits.getString("cpu") == null) {
        return ValidationResponse.reject(
          `Container '${containerName}' is missing CPU limit`,
          403
        ).toBuffer();
      }
    }

    if (settings.requireMemoryLimit) {
      if (limits == null || limits.getString("memory") == null) {
        return ValidationResponse.reject(
          `Container '${containerName}' is missing memory limit`,
          403
        ).toBuffer();
      }
    }
  }

  return ValidationResponse.accept().toBuffer();
}

// Validate the policy settings themselves
export function validate_settings(payload: ArrayBuffer): ArrayBuffer {
  const settings = JSON.parse(String.UTF8.decode(payload)) as JSON.Obj;
  if (settings == null) {
    return ValidationResponse.rejectSettings(
      "Cannot parse settings"
    ).toBuffer();
  }
  return ValidationResponse.acceptSettings().toBuffer();
}

function parseSettings(rawSettings: ArrayBuffer): Settings {
  const settings = new Settings();
  if (rawSettings.byteLength == 0) {
    return settings;
  }

  const parsed = JSON.parse(String.UTF8.decode(rawSettings)) as JSON.Obj;
  if (parsed == null) {
    return settings;
  }

  const requireCPU = parsed.getBool("requireCPULimit");
  if (requireCPU != null) {
    settings.requireCPULimit = requireCPU.valueOf();
  }

  const requireMem = parsed.getBool("requireMemoryLimit");
  if (requireMem != null) {
    settings.requireMemoryLimit = requireMem.valueOf();
  }

  return settings;
}
```

## Building the Policy

```bash
# Create build script in package.json
# Add to scripts section:
cat > build.sh <<'EOF'
#!/bin/bash

# Build the AssemblyScript policy to WebAssembly
npx asc assembly/index.ts \
  --target release \
  --binaryFile build/policy.wasm \
  --runtime stub \
  --exportRuntime false

echo "Build complete: build/policy.wasm"

# Annotate the policy with metadata
kwctl annotate \
  build/policy.wasm \
  --metadata-path metadata.yml \
  --output build/annotated-policy.wasm

echo "Annotation complete: build/annotated-policy.wasm"
EOF

chmod +x build.sh
./build.sh
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
  io.kubewarden.policy.title: require-resource-limits
  io.kubewarden.policy.description: Ensure all containers have CPU and memory limits
  io.kubewarden.policy.author: Platform Team
  io.kubewarden.policy.category: Resource management
  io.kubewarden.policy.severity: medium
```

## Testing the Policy

```bash
# Create test request for a pod with missing limits
cat > tests/data/no-limits-pod.json <<'EOF'
{
  "request": {
    "uid": "test-001",
    "kind": {"group": "", "version": "v1", "kind": "Pod"},
    "operation": "CREATE",
    "object": {
      "apiVersion": "v1",
      "kind": "Pod",
      "metadata": {"name": "test"},
      "spec": {
        "containers": [{
          "name": "app",
          "image": "nginx:1.25"
        }]
      }
    }
  }
}
EOF

# Test - should reject
kwctl run \
  build/annotated-policy.wasm \
  --request-path tests/data/no-limits-pod.json \
  --settings-json '{"requireCPULimit": true, "requireMemoryLimit": true}'

# Create test for a pod with limits
cat > tests/data/with-limits-pod.json <<'EOF'
{
  "request": {
    "uid": "test-002",
    "kind": {"group": "", "version": "v1", "kind": "Pod"},
    "operation": "CREATE",
    "object": {
      "spec": {
        "containers": [{
          "name": "app",
          "image": "nginx:1.25",
          "resources": {
            "limits": {"cpu": "500m", "memory": "512Mi"},
            "requests": {"cpu": "100m", "memory": "128Mi"}
          }
        }]
      }
    }
  }
}
EOF

# Test - should accept
kwctl run \
  build/annotated-policy.wasm \
  --request-path tests/data/with-limits-pod.json
```

## Deploying the Policy

```bash
# Push to OCI registry
kwctl push \
  build/annotated-policy.wasm \
  registry.example.com/kubewarden/require-resource-limits:v0.1.0
```

```yaml
# deploy.yaml
apiVersion: policies.kubewarden.io/v1
kind: ClusterAdmissionPolicy
metadata:
  name: require-resource-limits
spec:
  module: registry://registry.example.com/kubewarden/require-resource-limits:v0.1.0
  settings:
    requireCPULimit: true
    requireMemoryLimit: true
  rules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      resources: ["pods"]
      operations: ["CREATE", "UPDATE"]
  mutating: false
  mode: monitor
```

## Conclusion

AssemblyScript provides a TypeScript-like development experience for writing Kubewarden policies, making it accessible to frontend and full-stack developers who want to contribute to cluster security policies. While it requires more boilerplate than Rust for complex JSON manipulation, the familiar syntax and npm-based toolchain make it an excellent choice for teams already working with JavaScript/TypeScript ecosystems.
