# How to Write Custom Kubewarden Policies in Go - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Kubewarden, Go, Policy as Code, Kubernetes, WebAssembly, Admission Control, SUSE Rancher

Description: Learn how to write custom Kubewarden admission policies in Go compiled to WebAssembly using the Kubewarden Go SDK for type-safe Kubernetes object validation.

---

Writing Kubewarden policies in Go uses the familiar Go ecosystem and the Kubewarden Go SDK to inspect Kubernetes admission requests. The policy compiles to a WASM module that Kubewarden runs in a sandboxed environment.

---

## Prerequisites

```bash
# Install Go 1.21+

curl -Lo go.tar.gz https://go.dev/dl/go1.21.5.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf go.tar.gz
export PATH=$PATH:/usr/local/go/bin

# Install TinyGo (required for WASM compilation)
curl -Lo tinygo.tar.gz https://github.com/tinygo-org/tinygo/releases/download/v0.31.0/tinygo0.31.0.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf tinygo.tar.gz
export PATH=$PATH:/usr/local/tinygo/bin

# Install kwctl
curl -Lo kwctl https://github.com/kubewarden/kwctl/releases/latest/download/kwctl-linux-amd64
chmod +x kwctl && sudo mv kwctl /usr/local/bin/
```

---

## Step 1: Create a New Policy Project

```bash
# Use the Go policy template
go install github.com/nicholasgasior/gsfmt@latest

# Initialize the project
mkdir disallow-latest-tag && cd disallow-latest-tag
go mod init github.com/my-org/disallow-latest-tag

# Add Kubewarden SDK
go get github.com/kubewarden/policy-sdk-go
```

---

## Step 2: Write the Policy

```go
// main.go
package main

import (
    "encoding/json"
    "fmt"
    "strings"

    mapset "github.com/deckarep/golang-set/v2"
    kubewarden "github.com/kubewarden/policy-sdk-go"
    kubewarden_protocol "github.com/kubewarden/policy-sdk-go/protocol"
    corev1 "github.com/kubewarden/k8s-objects/api/core/v1"
)

func validate(payload []byte) ([]byte, error) {
    // Unmarshal the admission request
    validationRequest := kubewarden_protocol.ValidationRequest{}
    if err := json.Unmarshal(payload, &validationRequest); err != nil {
        return kubewarden.RejectRequest(
            kubewarden.Message(fmt.Sprintf("Cannot unmarshal request: %v", err)),
            kubewarden.Code(400),
        )
    }

    // Get the pod object
    pod := &corev1.Pod{}
    if err := json.Unmarshal(validationRequest.Request.Object.Raw, pod); err != nil {
        return kubewarden.RejectRequest(
            kubewarden.Message("Cannot unmarshal Pod object"),
            kubewarden.Code(400),
        )
    }

    // Check all containers for the :latest tag
    violations := []string{}
    for _, container := range pod.Spec.Containers {
        if strings.HasSuffix(container.Image, ":latest") || !strings.Contains(container.Image, ":") {
            violations = append(violations,
                fmt.Sprintf("Container '%s' uses the :latest tag or has no tag - use a specific version", container.Name))
        }
    }

    if len(violations) > 0 {
        return kubewarden.RejectRequest(
            kubewarden.Message(strings.Join(violations, "; ")),
            kubewarden.Code(400),
        )
    }

    return kubewarden.AcceptRequest()
}

// main is the entry point for the WASM module
func main() {}
```

---

## Step 3: Build the WASM Policy

```bash
# Compile to WASM using TinyGo
tinygo build \
  -o disallow-latest-tag.wasm \
  -target wasi \
  -no-debug .

# Verify the WASM file was created
ls -la disallow-latest-tag.wasm
```

---

## Step 4: Test the Policy

```bash
# Test against a pod using :latest tag (should be rejected)
cat > test-latest.json << EOF
{
  "request": {
    "operation": "CREATE",
    "kind": {"version":"v1","kind":"Pod"},
    "object": {
      "spec": {
        "containers": [{"name":"app","image":"nginx:latest"}]
      }
    }
  }
}
EOF

kwctl run disallow-latest-tag.wasm --request-path test-latest.json
```

---

## Step 5: Deploy the Policy

```bash
# Annotate the policy with metadata
kwctl annotate disallow-latest-tag.wasm \
  --metadata-path metadata.yaml \
  --output annotated-policy.wasm

# Push to OCI registry
kwctl push annotated-policy.wasm \
  ghcr.io/my-org/disallow-latest-tag:v0.1.0

# Create a ClusterAdmissionPolicy
kubectl apply -f - <<EOF
apiVersion: policies.kubewarden.io/v1
kind: ClusterAdmissionPolicy
metadata:
  name: disallow-latest-tag
spec:
  module: ghcr.io/my-org/disallow-latest-tag:v0.1.0
  rules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      resources: ["pods"]
      operations: ["CREATE", "UPDATE"]
  mutating: false
EOF
```

---

## Best Practices

- Use TinyGo's WASM compilation (not standard Go) - it produces much smaller WASM binaries.
- Write table-driven tests in Go to cover edge cases before building the WASM.
- Use `kwctl verify` to validate policy signatures before deploying to production clusters.
