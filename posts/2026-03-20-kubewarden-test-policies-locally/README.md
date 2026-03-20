# How to Test Kubewarden Policies Locally - Locally

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Kubewarden, Kubernetes, Policy, Testing, Kwctl

Description: Learn how to test Kubewarden policies locally using kwctl before deploying them to your cluster, ensuring they behave as expected without risking production disruptions.

## Introduction

Testing Kubewarden policies before deploying them to your cluster is essential for ensuring they behave correctly and won't accidentally block legitimate workloads. Kubewarden provides `kwctl`, a CLI tool that can run policy Wasm modules locally against simulated admission requests, making it easy to validate your policies in a development environment.

## Prerequisites

- `kwctl` CLI installed
- A Kubewarden policy (Wasm file or OCI reference)
- `kubectl` for generating test payloads from live clusters
- `jq` for JSON manipulation (optional but helpful)

## Installing kwctl

```bash
# Linux (amd64)

curl -LO https://github.com/kubewarden/kwctl/releases/latest/download/kwctl-linux-amd64
chmod +x kwctl-linux-amd64
sudo mv kwctl-linux-amd64 /usr/local/bin/kwctl

# macOS (arm64)
curl -LO https://github.com/kubewarden/kwctl/releases/latest/download/kwctl-darwin-arm64
chmod +x kwctl-darwin-arm64
sudo mv kwctl-darwin-arm64 /usr/local/bin/kwctl

# Verify installation
kwctl --version
```

## Understanding kwctl Test Payloads

A kwctl test payload is a JSON file that simulates a Kubernetes admission request. It contains:
- `request.uid`: Unique request ID
- `request.kind`: The resource type being admitted
- `request.operation`: CREATE, UPDATE, DELETE, or CONNECT
- `request.object`: The Kubernetes resource being validated
- `request.oldObject`: The previous state (for UPDATE operations)
- `settings`: Policy-specific configuration

## Creating Test Payloads

### Manual Test Payload

```json
// tests/valid-pod.json - A pod that should be ALLOWED
{
  "request": {
    "uid": "test-001",
    "kind": {
      "group": "",
      "version": "v1",
      "kind": "Pod"
    },
    "resource": {
      "group": "",
      "version": "v1",
      "resource": "pods"
    },
    "operation": "CREATE",
    "namespace": "production",
    "object": {
      "apiVersion": "v1",
      "kind": "Pod",
      "metadata": {
        "name": "valid-pod",
        "namespace": "production",
        "labels": {
          "app": "my-app",
          "team": "backend"
        }
      },
      "spec": {
        "containers": [
          {
            "name": "app",
            "image": "nginx:1.25.0",
            "resources": {
              "requests": {"cpu": "100m", "memory": "128Mi"},
              "limits": {"cpu": "500m", "memory": "512Mi"}
            },
            "securityContext": {
              "allowPrivilegeEscalation": false,
              "readOnlyRootFilesystem": true,
              "runAsNonRoot": true,
              "runAsUser": 1000
            }
          }
        ]
      }
    }
  }
}
```

```json
// tests/invalid-pod.json - A pod that should be DENIED
{
  "request": {
    "uid": "test-002",
    "kind": {"group": "", "version": "v1", "kind": "Pod"},
    "operation": "CREATE",
    "namespace": "production",
    "object": {
      "apiVersion": "v1",
      "kind": "Pod",
      "metadata": {"name": "privileged-pod"},
      "spec": {
        "containers": [
          {
            "name": "app",
            "image": "nginx:latest",
            "securityContext": {
              "privileged": true
            }
          }
        ]
      }
    }
  }
}
```

### Generating Payloads from Live Cluster Resources

```bash
# Get an existing pod and format it as a test payload
POD_JSON=$(kubectl get pod my-pod -n production -o json)

# Create the admission request wrapper
cat <<EOF > tests/existing-pod-request.json
{
  "request": {
    "uid": "test-from-cluster",
    "kind": {"group": "", "version": "v1", "kind": "Pod"},
    "operation": "CREATE",
    "namespace": "production",
    "object": $(echo "$POD_JSON")
  }
}
EOF

echo "Test payload created: tests/existing-pod-request.json"
```

## Running Tests with kwctl

### Basic Test Run

```bash
# Test a policy from OCI registry against a request
kwctl run \
  registry://ghcr.io/kubewarden/policies/pod-privileged:v0.2.0 \
  --request-path tests/invalid-pod.json

# Test a local Wasm file
kwctl run \
  ./build/policy.wasm \
  --request-path tests/valid-pod.json

# Test with specific settings
kwctl run \
  registry://ghcr.io/kubewarden/policies/host-namespaces-psp:v0.1.1 \
  --request-path tests/pod.json \
  --settings-json '{"hostPID": false, "hostIPC": false, "hostNetwork": false}'
```

### Testing Settings Validation

```bash
# Validate policy settings (separate from request validation)
kwctl run \
  --validate-settings \
  registry://ghcr.io/kubewarden/policies/allowed-image-repositories:v0.1.0 \
  --settings-json '{"allowedRegistries": ["registry.example.com"]}'

# Test invalid settings
kwctl run \
  --validate-settings \
  registry://ghcr.io/kubewarden/policies/allowed-image-repositories:v0.1.0 \
  --settings-json '{}'  # Empty settings - should fail validation
```

## Batch Testing with a Test Suite

Create a comprehensive test suite:

```bash
#!/bin/bash
# test-policy.sh - Run all tests for a policy

POLICY="registry://ghcr.io/kubewarden/policies/pod-privileged:v0.2.0"
TESTS_PASS=0
TESTS_FAIL=0

run_test() {
    local test_name="$1"
    local request_file="$2"
    local expected_accept="$3"  # true or false

    result=$(kwctl run "$POLICY" --request-path "$request_file" 2>&1)
    accepted=$(echo "$result" | grep -q "allowed: true" && echo "true" || echo "false")

    if [ "$accepted" = "$expected_accept" ]; then
        echo "PASS: $test_name"
        TESTS_PASS=$((TESTS_PASS + 1))
    else
        echo "FAIL: $test_name (expected accepted=$expected_accept, got $accepted)"
        echo "      Output: $result"
        TESTS_FAIL=$((TESTS_FAIL + 1))
    fi
}

echo "Running policy tests..."
echo "========================"

# Test cases
run_test "Allow normal pod" tests/valid-pod.json "true"
run_test "Deny privileged pod" tests/privileged-pod.json "false"
run_test "Allow pod without securityContext" tests/no-security-context.json "true"

echo "========================"
echo "Results: ${TESTS_PASS} passed, ${TESTS_FAIL} failed"

exit $TESTS_FAIL
```

## Testing UPDATE Operations

For validating UPDATE behavior (existing resources being modified):

```json
// tests/update-request.json
{
  "request": {
    "uid": "update-test-001",
    "kind": {"group": "", "version": "v1", "kind": "Pod"},
    "operation": "UPDATE",
    "namespace": "production",
    "object": {
      "spec": {
        "containers": [
          {"name": "app", "image": "nginx:1.26.0"}
        ]
      }
    },
    "oldObject": {
      "spec": {
        "containers": [
          {"name": "app", "image": "nginx:1.25.0"}
        ]
      }
    }
  }
}
```

```bash
# Test the UPDATE operation
kwctl run \
  ./build/policy.wasm \
  --request-path tests/update-request.json
```

## Inspecting Policy Metadata

```bash
# View policy metadata before deploying
kwctl inspect \
  registry://ghcr.io/kubewarden/policies/pod-privileged:v0.2.0

# Get the full policy manifest
kwctl manifest \
  registry://ghcr.io/kubewarden/policies/pod-privileged:v0.2.0 \
  --type ClusterAdmissionPolicy
```

## CI/CD Integration

```yaml
# .github/workflows/test-policy.yml
name: Test Kubewarden Policy

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install kwctl
        run: |
          curl -LO https://github.com/kubewarden/kwctl/releases/latest/download/kwctl-linux-amd64
          chmod +x kwctl-linux-amd64
          sudo mv kwctl-linux-amd64 /usr/local/bin/kwctl

      - name: Build policy
        run: make build

      - name: Run policy tests
        run: ./test-policy.sh
```

## Conclusion

Local testing with `kwctl` is an essential practice for Kubewarden policy development. By creating comprehensive test suites that cover both allowed and denied scenarios, you can confidently deploy policies knowing they will behave as intended. Integrating `kwctl` tests into your CI/CD pipeline ensures that policy changes are validated automatically before they reach your cluster, maintaining the reliability of your admission control system.
