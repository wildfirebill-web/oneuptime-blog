# How to Write Custom Kubewarden Policies in AssemblyScript

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Kubewarden, AssemblyScript, TypeScript, Policy as Code, Kubernetes, WebAssembly, Admission Control

Description: Learn how to write custom Kubewarden admission policies in AssemblyScript — a TypeScript-like language that compiles to WebAssembly — for familiar JavaScript-style policy authoring.

---

AssemblyScript allows developers familiar with TypeScript to write Kubewarden policies without learning Rust or Go. It compiles directly to WebAssembly and integrates with the Kubewarden SDK.

---

## Prerequisites

```bash
# Install Node.js 18+
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo bash
sudo apt-get install -y nodejs

# Install the Kubewarden AssemblyScript scaffolding tool
npm install -g @kubewarden/as-policy-template
```

---

## Step 1: Create a Policy Project

```bash
# Scaffold a new policy
npx @kubewarden/as-policy-template my-policy

cd my-policy
npm install
```

---

## Step 2: Write the Policy Logic

Edit `assembly/index.ts` to implement your policy:

```typescript
// assembly/index.ts
import { JSON } from "assemblyscript-json/assembly";
import { validateSettings, Settings } from "./settings";

// Main validation function — called for each admission request
export function validate(payload: string): string {
  const payloadObj = JSON.parse(payload) as JSON.Obj;

  // Extract the request object
  const request = payloadObj.getObj("request");
  if (request === null) {
    return JSON.stringify({"accepted": true});
  }

  const object = request.getObj("object");
  if (object === null) {
    return JSON.stringify({"accepted": true});
  }

  // Get pod spec
  const spec = object.getObj("spec");
  if (spec === null) {
    return JSON.stringify({"accepted": true});
  }

  // Get containers
  const containers = spec.getArr("containers");
  if (containers === null) {
    return JSON.stringify({"accepted": true});
  }

  // Check each container has a non-root user
  const violations: string[] = [];
  for (let i = 0; i < containers.valueOf().length; i++) {
    const container = containers.valueOf()[i] as JSON.Obj;
    const containerName = container.getString("name");
    const securityContext = container.getObj("securityContext");

    if (securityContext === null) {
      violations.push(
        `Container '${containerName !== null ? containerName.valueOf() : "unknown"}' must have a securityContext`
      );
      continue;
    }

    const runAsNonRoot = securityContext.getBool("runAsNonRoot");
    if (runAsNonRoot === null || !runAsNonRoot.valueOf()) {
      violations.push(
        `Container '${containerName !== null ? containerName.valueOf() : "unknown"}' must set runAsNonRoot: true`
      );
    }
  }

  if (violations.length > 0) {
    return JSON.stringify({
      "accepted": false,
      "message": violations.join("; ")
    });
  }

  return JSON.stringify({"accepted": true});
}
```

---

## Step 3: Build the WASM Policy

```bash
# Compile to WASM
npm run build

# The WASM file is generated at:
ls build/release.wasm
```

---

## Step 4: Test Locally

```bash
# Install kwctl
curl -Lo kwctl https://github.com/kubewarden/kwctl/releases/latest/download/kwctl-linux-amd64
chmod +x kwctl && sudo mv kwctl /usr/local/bin/

# Create a test request (pod without securityContext)
cat > test-no-security-context.json << EOF
{
  "request": {
    "operation": "CREATE",
    "object": {
      "spec": {
        "containers": [{"name": "app", "image": "nginx:1.24"}]
      }
    }
  }
}
EOF

# Run the policy test
kwctl run build/release.wasm --request-path test-no-security-context.json
# Expected: rejected — no securityContext
```

---

## Step 5: Annotate and Deploy

```bash
# Annotate with policy metadata
kwctl annotate build/release.wasm \
  --metadata-path metadata.yaml \
  --output annotated.wasm

# Push to a registry
kwctl push annotated.wasm \
  ghcr.io/my-org/require-non-root:v0.1.0

# Deploy via Kubewarden
kubectl apply -f - <<EOF
apiVersion: policies.kubewarden.io/v1
kind: ClusterAdmissionPolicy
metadata:
  name: require-non-root
spec:
  module: ghcr.io/my-org/require-non-root:v0.1.0
  rules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      resources: ["pods"]
      operations: ["CREATE"]
  mutating: false
EOF
```

---

## Best Practices

- AssemblyScript's JSON handling requires careful null checking — always handle `null` returns from JSON accessors.
- Write tests using the AssemblyScript test runner before compiling to WASM.
- For complex policies with many rules, prefer Rust or Go which have richer SDK support.
