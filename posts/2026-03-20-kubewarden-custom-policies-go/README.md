# How to Write Custom Kubewarden Policies in Go

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Kubewarden, Kubernetes, Policy, Go, WebAssembly

Description: Learn how to write custom Kubewarden admission control policies in Go using the TinyGo compiler and Kubewarden Go SDK for WebAssembly compilation.

## Introduction

Go is a popular choice for writing Kubewarden policies, especially for teams already familiar with Kubernetes tooling (which is written in Go). Kubewarden provides a Go SDK that enables you to write policies in idiomatic Go, compile them to WebAssembly using TinyGo, and deploy them as Kubewarden admission policies.

## Prerequisites

- Go 1.21 or later
- TinyGo 0.30 or later (for Wasm compilation)
- `kwctl` CLI installed
- Basic Go programming knowledge

## Setting Up the Development Environment

```bash
# Install TinyGo

# macOS
brew tap tinygo-org/tools
brew install tinygo

# Linux (download from releases)
curl -LO https://github.com/tinygo-org/tinygo/releases/download/v0.30.0/tinygo0.30.0.linux-amd64.tar.gz
tar -xzf tinygo0.30.0.linux-amd64.tar.gz
sudo mv tinygo /usr/local/tinygo
export PATH=$PATH:/usr/local/tinygo/bin

# Verify TinyGo installation
tinygo version

# Install the Go policy template
go install github.com/kubewarden/kubewarden-controller/tools/create-policy@latest
```

## Creating a New Go Policy

```bash
# Create a new policy from template
mkdir my-go-policy && cd my-go-policy

# Initialize the Go module
go mod init github.com/my-org/my-go-policy

# Install the Kubewarden Go SDK
go get github.com/kubewarden/policy-sdk-go
```

## Project Structure

```text
my-go-policy/
├── go.mod
├── go.sum
├── main.go             # Main policy logic
├── settings.go         # Policy settings
├── metadata.yml        # Policy metadata
└── Makefile
```

## Writing the Policy

### Main Policy File

```go
// main.go - Kubewarden policy in Go
package main

import (
    "encoding/json"
    "fmt"

    corev1 "github.com/kubewarden/k8s-objects/api/core/v1"
    kubewarden "github.com/kubewarden/policy-sdk-go"
    kubewarden_protocol "github.com/kubewarden/policy-sdk-go/protocol"
)

// Settings holds the policy configuration
type Settings struct {
    // Minimum number of replicas required
    MinReplicas int32 `json:"minReplicas"`
    // Maximum number of replicas allowed
    MaxReplicas int32 `json:"maxReplicas"`
}

func (s *Settings) Valid() (bool, error) {
    if s.MinReplicas < 1 {
        return false, fmt.Errorf("minReplicas must be at least 1")
    }
    if s.MaxReplicas < s.MinReplicas {
        return false, fmt.Errorf("maxReplicas must be >= minReplicas")
    }
    return true, nil
}

// main is required for TinyGo compilation but never called
func main() {}

// validate is the entry point called by the Kubewarden host
//
//export validate
func validate() {
    // Read the validation request payload
    payload, err := kubewarden.Host.Read()
    if err != nil {
        kubewarden.RejectRequest(
            kubewarden.Message(fmt.Sprintf("Cannot read payload: %v", err)),
            kubewarden.Code(400))
        return
    }

    // Parse the validation request
    validationRequest := kubewarden_protocol.ValidationRequest{}
    if err = json.Unmarshal(payload, &validationRequest); err != nil {
        kubewarden.RejectRequest(
            kubewarden.Message(fmt.Sprintf("Cannot parse request: %v", err)),
            kubewarden.Code(400))
        return
    }

    // Parse policy settings
    settings := Settings{}
    if err = json.Unmarshal(validationRequest.Settings, &settings); err != nil {
        kubewarden.RejectRequest(
            kubewarden.Message(fmt.Sprintf("Cannot parse settings: %v", err)),
            kubewarden.Code(400))
        return
    }

    // Validate the Deployment object
    deployment := &corev1.Pod{}
    if err = json.Unmarshal(validationRequest.Request.Object, deployment); err != nil {
        // Not a Pod, accept the request
        kubewarden.AcceptRequest()
        return
    }

    // Apply policy logic
    if err = validateDeployment(deployment, &settings); err != nil {
        kubewarden.RejectRequest(
            kubewarden.Message(err.Error()),
            kubewarden.NoCode)
        return
    }

    kubewarden.AcceptRequest()
}

// validateDeployment checks the replica count constraints
func validateDeployment(pod *corev1.Pod, settings *Settings) error {
    // Policy logic goes here
    // This example checks container image tags
    for _, container := range pod.Spec.Containers {
        if container.Image == nil {
            continue
        }
        image := *container.Image
        if isLatestTag(image) {
            return fmt.Errorf(
                "container %q uses 'latest' tag. Use a specific version tag",
                container.Name)
        }
    }
    return nil
}

func isLatestTag(image string) bool {
    // No tag specified defaults to latest
    if !containsColon(image) {
        return true
    }
    // Check for explicit latest tag
    return hasSuffix(image, ":latest")
}

func containsColon(s string) bool {
    for _, c := range s {
        if c == ':' {
            return true
        }
    }
    return false
}

func hasSuffix(s, suffix string) bool {
    if len(s) < len(suffix) {
        return false
    }
    return s[len(s)-len(suffix):] == suffix
}

// validate_settings is called to validate policy settings
//
//export validate_settings
func validateSettings() {
    payload, err := kubewarden.Host.Read()
    if err != nil {
        kubewarden.RejectSettings(
            kubewarden.Message(fmt.Sprintf("Cannot read settings: %v", err)))
        return
    }

    settings := Settings{}
    if err = json.Unmarshal(payload, &settings); err != nil {
        kubewarden.RejectSettings(
            kubewarden.Message(fmt.Sprintf("Cannot parse settings: %v", err)))
        return
    }

    if valid, err := settings.Valid(); !valid {
        kubewarden.RejectSettings(kubewarden.Message(err.Error()))
        return
    }

    kubewarden.AcceptSettings()
}
```

## Building the Policy

```bash
# Create the Makefile
cat > Makefile <<'EOF'
CLUSTER_POLICY_MODULE=policy.wasm

.PHONY: build
build:
	tinygo build \
		-target wasi \
		-gc=leaking \
		-o $(CLUSTER_POLICY_MODULE) \
		.

.PHONY: annotate
annotate: build
	kwctl annotate \
		$(CLUSTER_POLICY_MODULE) \
		--metadata-path metadata.yml \
		--output annotated-$(CLUSTER_POLICY_MODULE)

.PHONY: test
test:
	go test ./...

.PHONY: clean
clean:
	rm -f *.wasm
EOF

# Build the policy
make build

# Annotate with metadata
make annotate
```

## Creating Policy Metadata

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
  io.kubewarden.policy.title: no-latest-tag
  io.kubewarden.policy.description: Reject pods using 'latest' image tag
  io.kubewarden.policy.author: My Team
  io.kubewarden.policy.url: https://github.com/my-org/my-go-policy
  io.kubewarden.policy.category: Best practices
  io.kubewarden.policy.severity: medium
```

## Testing with kwctl

```bash
# Create a test request payload
cat > test-request.json <<EOF
{
  "settings": {
    "minReplicas": 1,
    "maxReplicas": 10
  },
  "request": {
    "uid": "test-001",
    "kind": {"group": "", "version": "v1", "kind": "Pod"},
    "operation": "CREATE",
    "object": {
      "apiVersion": "v1",
      "kind": "Pod",
      "metadata": {"name": "test"},
      "spec": {
        "containers": [{"name": "app", "image": "nginx:latest"}]
      }
    }
  }
}
EOF

# Test the policy
kwctl run annotated-policy.wasm --request-path test-request.json
```

## Deploying the Go Policy

```bash
# Push to OCI registry
kwctl push \
  annotated-policy.wasm \
  registry.example.com/kubewarden/no-latest-tag:v0.1.0
```

```yaml
# deploy-go-policy.yaml
apiVersion: policies.kubewarden.io/v1
kind: ClusterAdmissionPolicy
metadata:
  name: no-latest-tag-go
spec:
  module: registry://registry.example.com/kubewarden/no-latest-tag:v0.1.0
  settings:
    minReplicas: 1
    maxReplicas: 10
  rules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      resources: ["pods"]
      operations: ["CREATE", "UPDATE"]
  mutating: false
  mode: protect
```

## Conclusion

Writing Kubewarden policies in Go enables Kubernetes-native teams to leverage their existing Go expertise for admission control. The TinyGo compiler makes it straightforward to compile Go code to WebAssembly, and the Kubewarden Go SDK provides a clean API for handling admission requests. The combination of Go's familiar patterns, TinyGo's WebAssembly output, and Kubewarden's policy framework gives you a productive and powerful platform for building custom security policies.
