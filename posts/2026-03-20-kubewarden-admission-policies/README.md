# How to Create Kubewarden Admission Policies

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Kubewarden, Kubernetes, Policy, Security, Admission Control

Description: Learn how to create namespace-scoped Kubewarden AdmissionPolicy resources to enforce security and compliance rules on Kubernetes resources within specific namespaces.

## Introduction

Kubewarden AdmissionPolicies are namespace-scoped resources that intercept and evaluate Kubernetes API requests before they are persisted. Unlike ClusterAdmissionPolicies (which are cluster-wide), AdmissionPolicies apply only to resources within the namespace where they are created, making them ideal for per-team or per-application policy enforcement.

This guide covers creating, configuring, and managing namespace-scoped AdmissionPolicies.

## Prerequisites

- Kubewarden installed on your cluster
- `kubectl` access with namespace admin permissions
- Basic understanding of Kubernetes admission webhooks

## Understanding AdmissionPolicy vs ClusterAdmissionPolicy

| Feature | AdmissionPolicy | ClusterAdmissionPolicy |
|---------|----------------|------------------------|
| Scope | Single namespace | All namespaces |
| Created by | Namespace admin | Cluster admin |
| Typical use | Team/app policies | Platform-wide policies |
| Can target namespace resources | Yes | Yes |
| Can target cluster resources | No | Yes |

## Creating a Basic AdmissionPolicy

### Preventing Privileged Pods in a Namespace

```yaml
# no-privileged-pods-policy.yaml
apiVersion: policies.kubewarden.io/v1
kind: AdmissionPolicy
metadata:
  name: no-privileged-pods
  # Only applies within this namespace
  namespace: production
spec:
  # Wasm module URI from Kubewarden policy hub
  module: registry://ghcr.io/kubewarden/policies/pod-privileged:v0.2.0

  # Kubernetes resources this policy watches
  rules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      resources: ["pods"]
      operations:
        # Evaluate CREATE and UPDATE operations
        - CREATE
        - UPDATE

  # false = validation policy (doesn't modify resources)
  mutating: false

  # Fail closed if policy evaluation errors occur
  failurePolicy: Fail

  # Policy is active (not just monitoring)
  mode: protect
```

```bash
# Apply the policy
kubectl apply -f no-privileged-pods-policy.yaml

# Check the policy status
kubectl get admissionpolicy no-privileged-pods -n production

# Wait for the policy to become active
kubectl wait admissionpolicy no-privileged-pods \
  -n production \
  --for=condition=PolicyActive \
  --timeout=60s
```

### Testing the Policy

```bash
# Try to create a privileged pod (should be DENIED)
kubectl apply -n production -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: privileged-test
spec:
  containers:
    - name: test
      image: nginx:latest
      securityContext:
        privileged: true  # This should be blocked!
EOF

# Expected output: Error from server: ...
# admission webhook denied the request

# Try to create a non-privileged pod (should SUCCEED)
kubectl apply -n production -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: normal-test
spec:
  containers:
    - name: test
      image: nginx:latest
EOF
```

## Creating Policies with Custom Settings

### Requiring Specific Labels

```yaml
# require-labels-policy.yaml
apiVersion: policies.kubewarden.io/v1
kind: AdmissionPolicy
metadata:
  name: require-team-label
  namespace: production
spec:
  module: registry://ghcr.io/kubewarden/policies/k8s-objects:v1.3.0

  settings:
    # Policy-specific configuration
    mandatory_labels:
      - team
      - cost-center
      - app

  rules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      resources: ["pods"]
      operations:
        - CREATE
  mutating: false
  failurePolicy: Fail
  mode: protect
```

### Enforcing Resource Requests and Limits

```yaml
# resource-limits-policy.yaml
apiVersion: policies.kubewarden.io/v1
kind: AdmissionPolicy
metadata:
  name: require-resource-limits
  namespace: development
spec:
  module: registry://ghcr.io/kubewarden/policies/require-resources:v0.1.0

  settings:
    # Require both requests and limits for all containers
    memory:
      requireLimit: true
      requireRequest: true
    cpu:
      requireLimit: true
      requireRequest: true

  rules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      resources: ["pods"]
      operations:
        - CREATE
        - UPDATE
  mutating: false
  failurePolicy: Fail
  mode: protect
```

## Using Audit Mode for Non-Disruptive Testing

Before enforcing a policy, run it in audit mode to see what would be blocked:

```yaml
# policy-audit-mode.yaml
apiVersion: policies.kubewarden.io/v1
kind: AdmissionPolicy
metadata:
  name: audit-no-latest-tag
  namespace: production
spec:
  module: registry://ghcr.io/kubewarden/policies/verify-image-signatures:v0.1.0

  settings:
    signatures:
      - image: "*"
        anyOf:
          - kind: githubAction

  rules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      resources: ["pods"]
      operations:
        - CREATE
        - UPDATE
  mutating: false

  # Audit mode: policy evaluates but doesn't block
  # Violations are logged but requests are allowed
  mode: monitor
```

```bash
# Apply in monitor mode first
kubectl apply -f policy-audit-mode.yaml

# Check events to see what would have been blocked
kubectl get events -n production \
  --field-selector reason=PolicyViolation
```

## Switching Policy Mode

```bash
# Switch from monitor to protect mode
kubectl patch admissionpolicy audit-no-latest-tag \
  -n production \
  --type=merge \
  -p '{"spec":{"mode":"protect"}}'

# Switch back to monitor if issues are discovered
kubectl patch admissionpolicy audit-no-latest-tag \
  -n production \
  --type=merge \
  -p '{"spec":{"mode":"monitor"}}'
```

## Checking Policy Status

```bash
# List all admission policies in a namespace
kubectl get admissionpolicies -n production

# Get detailed policy status
kubectl describe admissionpolicy no-privileged-pods -n production

# Check conditions
kubectl get admissionpolicy no-privileged-pods -n production \
  -o jsonpath='{.status.conditions}' | python3 -m json.tool
```

Policy status conditions:
- `PolicyActive`: Policy is active and enforcing
- `PolicyUniquelyReachable`: No conflicts with other policies

## Deleting a Policy

```bash
# Delete the admission policy
kubectl delete admissionpolicy no-privileged-pods -n production

# Verify deletion
kubectl get admissionpolicies -n production
```

## Conclusion

Kubewarden AdmissionPolicies provide namespace-scoped policy enforcement that can be managed by namespace owners without requiring cluster-admin privileges. By combining validation and mutation policies in audit mode first, then switching to protect mode, you can safely roll out new policies without disrupting existing workloads. The granular namespace scoping makes AdmissionPolicies ideal for implementing team-specific security requirements while maintaining a shared cluster infrastructure.
