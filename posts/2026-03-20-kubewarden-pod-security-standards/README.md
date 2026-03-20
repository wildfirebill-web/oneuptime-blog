# How to Set Up Kubewarden for Pod Security Standards

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Kubewarden, Kubernetes, PodSecurity, Standards, Compliance

Description: Learn how to implement Kubernetes Pod Security Standards (Privileged, Baseline, and Restricted) using Kubewarden policies for fine-grained admission control.

## Introduction

Kubernetes Pod Security Standards (PSS) define three security profiles for pod workloads:
- **Privileged**: No restrictions, for trusted workloads
- **Baseline**: Prevents known privilege escalations
- **Restricted**: Follows best practices for hardened security

While Kubernetes has built-in Pod Security Admission (PSA), Kubewarden provides more granular control — you can implement each check of the security standards individually, add exceptions per workload, combine them with custom policies, and get detailed violation messages.

## Prerequisites

- Kubewarden installed on the cluster
- `kubectl` with cluster-admin access

## Understanding Pod Security Standards Checks

The Restricted profile includes all Baseline checks plus additional requirements:

### Baseline Profile Checks
- No privileged containers
- No host namespaces (hostPID, hostIPC, hostNetwork)
- No host path volumes
- No host ports
- Limited AppArmor profiles
- Restricted seccomp profiles
- No privileged sysctls

### Restricted Profile Adds
- No privilege escalation
- Non-root user required
- Seccomp profile required
- Capabilities must be dropped to ALL

## Implementing the Baseline Profile

```yaml
# kubewarden-baseline.yaml - Complete Baseline PSS implementation
---
# Block privileged containers
apiVersion: policies.kubewarden.io/v1
kind: ClusterAdmissionPolicy
metadata:
  name: pss-baseline-no-privileged
  labels:
    pss-profile: baseline
spec:
  module: registry://ghcr.io/kubewarden/policies/pod-privileged:v0.2.0
  rules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      resources: ["pods"]
      operations: ["CREATE", "UPDATE"]
  mutating: false
  mode: protect
  namespaceSelector:
    matchExpressions:
      - key: pod-security.kubernetes.io/enforce
        operator: In
        values: ["baseline", "restricted"]
---
# Block host namespaces
apiVersion: policies.kubewarden.io/v1
kind: ClusterAdmissionPolicy
metadata:
  name: pss-baseline-no-host-namespaces
  labels:
    pss-profile: baseline
spec:
  module: registry://ghcr.io/kubewarden/policies/host-namespaces-psp:v0.1.1
  settings:
    hostPID: false
    hostIPC: false
    hostNetwork: false
  rules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      resources: ["pods"]
      operations: ["CREATE", "UPDATE"]
  mutating: false
  mode: protect
  namespaceSelector:
    matchExpressions:
      - key: pod-security.kubernetes.io/enforce
        operator: In
        values: ["baseline", "restricted"]
---
# Restrict volume types
apiVersion: policies.kubewarden.io/v1
kind: ClusterAdmissionPolicy
metadata:
  name: pss-baseline-allowed-volumes
  labels:
    pss-profile: baseline
spec:
  module: registry://ghcr.io/kubewarden/policies/volumes-psp:v0.2.0
  settings:
    allowedTypes:
      - configMap
      - csi
      - downwardAPI
      - emptyDir
      - ephemeral
      - hostPath     # Allowed in baseline but not restricted
      - persistentVolumeClaim
      - projected
      - secret
  rules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      resources: ["pods"]
      operations: ["CREATE", "UPDATE"]
  mutating: false
  mode: protect
  namespaceSelector:
    matchExpressions:
      - key: pod-security.kubernetes.io/enforce
        operator: In
        values: ["baseline"]
```

## Implementing the Restricted Profile

```yaml
# kubewarden-restricted.yaml - Complete Restricted PSS implementation
---
# Require non-root user
apiVersion: policies.kubewarden.io/v1
kind: ClusterAdmissionPolicy
metadata:
  name: pss-restricted-non-root
  labels:
    pss-profile: restricted
spec:
  module: registry://ghcr.io/kubewarden/policies/user-group-psp:v0.4.0
  settings:
    run_as_user:
      rule: "MustRunAsNonRoot"
  rules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      resources: ["pods"]
      operations: ["CREATE", "UPDATE"]
  mutating: false
  mode: protect
  namespaceSelector:
    matchLabels:
      pod-security.kubernetes.io/enforce: restricted
---
# Require dropping ALL capabilities
apiVersion: policies.kubewarden.io/v1
kind: ClusterAdmissionPolicy
metadata:
  name: pss-restricted-capabilities
  labels:
    pss-profile: restricted
spec:
  module: registry://ghcr.io/kubewarden/policies/capabilities-psp:v0.2.0
  settings:
    allowed_capabilities: []
    required_drop_capabilities:
      - ALL
  rules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      resources: ["pods"]
      operations: ["CREATE", "UPDATE"]
  mutating: false
  mode: protect
  namespaceSelector:
    matchLabels:
      pod-security.kubernetes.io/enforce: restricted
---
# Require seccomp profile
apiVersion: policies.kubewarden.io/v1
kind: ClusterAdmissionPolicy
metadata:
  name: pss-restricted-seccomp
  labels:
    pss-profile: restricted
spec:
  module: registry://ghcr.io/kubewarden/policies/allowed-seccomp-profiles-psp:v0.1.8
  settings:
    allowed_profiles:
      - "runtime/default"
      - "localhost/*"
  rules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      resources: ["pods"]
      operations: ["CREATE", "UPDATE"]
  mutating: false
  mode: protect
  namespaceSelector:
    matchLabels:
      pod-security.kubernetes.io/enforce: restricted
```

## Labeling Namespaces for PSS

Label namespaces to indicate which PSS profile applies:

```bash
# Apply baseline to the development namespace
kubectl label namespace development \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/warn=restricted

# Apply restricted to the production namespace
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=restricted

# Exempt system namespaces (they need privileged access)
kubectl label namespace kube-system \
  pod-security.kubernetes.io/enforce=privileged

kubectl label namespace kubewarden \
  pod-security.kubernetes.io/enforce=privileged
```

## Applying All Policies at Once

```bash
# Apply all PSS policies
kubectl apply -f kubewarden-baseline.yaml
kubectl apply -f kubewarden-restricted.yaml

# Wait for all policies to become active
kubectl wait clusteradmissionpolicies \
  --all \
  --for=condition=PolicyActive \
  --timeout=120s

# Verify all PSS policies are active
kubectl get clusteradmissionpolicies \
  -l 'pss-profile in (baseline, restricted)'
```

## Testing PSS Compliance

```bash
# Test a restricted-profile non-compliant pod
kubectl apply -n production -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: test-non-compliant
spec:
  containers:
    - name: app
      image: nginx:1.25.0
      securityContext:
        runAsRoot: true  # Should be blocked in restricted namespaces
EOF

# Test a compliant pod
kubectl apply -n production -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: test-compliant
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: app
      image: nginx:1.25.0
      securityContext:
        allowPrivilegeEscalation: false
        capabilities:
          drop: ["ALL"]
EOF
```

## Monitoring PSS Policy Compliance

```bash
# Check for PSS policy violations
kubectl get events -A \
  -o jsonpath='{range .items[?(@.reason=="PolicyViolation")]}{.namespace}/{.involvedObject.name}: {.message}{"\n"}{end}'

# Find all pods that would violate restricted profile
for ns in $(kubectl get namespaces -o jsonpath='{.items[*].metadata.name}'); do
  kubectl get pods -n "$ns" \
    -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}' 2>/dev/null
done
```

## Conclusion

Implementing Kubernetes Pod Security Standards with Kubewarden gives you more control than the built-in PSA admission controller. By deploying individual Kubewarden policies for each check in the Baseline and Restricted profiles, you get detailed violation messages, per-check exemptions, and the ability to combine PSS policies with your own custom security requirements. The namespace-label-based scoping mirrors the PSA interface, making it easy to migrate from or complement the built-in admission controller.
