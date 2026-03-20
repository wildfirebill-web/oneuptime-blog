# How to Test Kubewarden Policies Locally

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Kubewarden, Testing, Policy as Code, Kubernetes, Kwctl, SUSE Rancher, Quality Assurance

Description: Learn how to test Kubewarden admission policies locally using kwctl and the bats testing framework before deploying them to a Kubernetes cluster.

---

Testing Kubewarden policies locally saves time by catching policy logic errors before they affect a running cluster. The `kwctl` tool can run any Kubewarden policy against synthetic admission requests without a Kubernetes cluster.

---

## Tools Required

```bash
# Install kwctl

curl -Lo kwctl https://github.com/kubewarden/kwctl/releases/latest/download/kwctl-linux-amd64
chmod +x kwctl
sudo mv kwctl /usr/local/bin/

# Verify installation
kwctl --version

# Optional: Install bats for structured test suites
sudo apt-get install -y bats
```

---

## Step 1: Run a Policy Against a Test Request

Pull a policy from the Kubewarden Policy Hub and test it:

```bash
# Pull the policy locally
kwctl pull registry://ghcr.io/kubewarden/policies/pod-privileged:v0.2.5

# Create a test request - a pod requesting privileged mode
cat > test-privileged-pod.json << EOF
{
  "uid": "test-001",
  "kind": {"group":"","version":"v1","kind":"Pod"},
  "resource": {"group":"","version":"v1","resource":"pods"},
  "operation": "CREATE",
  "object": {
    "apiVersion": "v1",
    "kind": "Pod",
    "metadata": {"name": "test-pod"},
    "spec": {
      "containers": [{
        "name": "test",
        "image": "nginx:1.24",
        "securityContext": {
          "privileged": true
        }
      }]
    }
  }
}
EOF

# Run the policy
kwctl run \
  registry://ghcr.io/kubewarden/policies/pod-privileged:v0.2.5 \
  --request-path test-privileged-pod.json

# Expected output: {"uid":"test-001","allowed":false,"status":{"message":"..."}}
```

---

## Step 2: Test a Policy with Settings

Some policies accept settings. Provide them as a JSON file:

```bash
# Create settings
cat > settings.json << EOF
{
  "required_labels": ["app", "team", "version"]
}
EOF

# Run policy with settings
kwctl run \
  registry://ghcr.io/kubewarden/policies/safe-labels:v0.1.0 \
  --request-path test-pod.json \
  --settings-path settings.json
```

---

## Step 3: Create a Structured Test Suite with bats

`bats` (Bash Automated Testing System) provides organized test suites:

```bash
# tests/test_pod_privileged.bats
#!/usr/bin/env bats

POLICY_PATH="annotated-policy.wasm"

@test "reject privileged container" {
  run kwctl run $POLICY_PATH \
    --request-path tests/fixtures/privileged-pod.json
  [ "$status" -eq 0 ]
  echo "$output" | grep -q '"allowed":false'
}

@test "accept non-privileged container" {
  run kwctl run $POLICY_PATH \
    --request-path tests/fixtures/normal-pod.json
  [ "$status" -eq 0 ]
  echo "$output" | grep -q '"allowed":true'
}

@test "accept when no securityContext defined" {
  run kwctl run $POLICY_PATH \
    --request-path tests/fixtures/no-security-context.json
  [ "$status" -eq 0 ]
  echo "$output" | grep -q '"allowed":true'
}
```

Run the tests:

```bash
bats tests/test_pod_privileged.bats
```

---

## Step 4: Validate Policy Settings

```bash
# Test that invalid settings are rejected
cat > invalid-settings.json << EOF
{
  "maxReplicas": "not-a-number"
}
EOF

kwctl run my-policy.wasm \
  --settings-path invalid-settings.json \
  --validate-settings

# Should output validation error
```

---

## Step 5: Test Mutation Policies

For policies that mutate resources, verify the mutated output:

```bash
# Run mutation policy
kwctl run mutating-policy.wasm \
  --request-path test-pod.json

# Extract the patch from the response
kwctl run mutating-policy.wasm \
  --request-path test-pod.json \
  | jq -r '.patch' | base64 -d | jq .
```

---

## CI Integration

Add policy tests to your CI pipeline:

```yaml
# .github/workflows/test-policies.yml
- name: Test Kubewarden policies
  run: |
    curl -Lo kwctl https://github.com/kubewarden/kwctl/releases/latest/download/kwctl-linux-amd64
    chmod +x kwctl && sudo mv kwctl /usr/local/bin/
    bats tests/
```

---

## Best Practices

- Test both accepting (allowed) and rejecting (denied) scenarios for every policy.
- Include edge cases: empty specs, missing fields, multiple containers.
- Run policy tests in CI before any policy changes are merged to the main branch.
