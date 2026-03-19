# How to Configure Pod Security Admission in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Pod Security, Security, Permissions, RBAC

Description: A practical guide to configuring Pod Security Admission (PSA) in Rancher to enforce security standards for pod workloads.

Pod Security Admission (PSA) replaced Pod Security Policies (PSP) starting in Kubernetes 1.25. PSA enforces the Pod Security Standards at the namespace level, controlling what security contexts and capabilities pods can use. Rancher provides tools to configure PSA across your clusters. This guide walks through setting up PSA effectively.

## Prerequisites

- Rancher v2.7+ managing Kubernetes 1.25+ clusters
- Administrator or cluster owner access
- Understanding of the three Pod Security Standards: Privileged, Baseline, and Restricted

## Understanding Pod Security Standards

Kubernetes defines three security levels:

- **Privileged**: Unrestricted. Allows all pod configurations. Use only for system-level workloads.
- **Baseline**: Prevents known privilege escalations. Allows most standard workloads. A good starting point.
- **Restricted**: Most restrictive. Enforces current pod hardening best practices. Ideal for untrusted workloads.

Each level can be applied in three modes:

- **enforce**: Violations prevent pods from being created.
- **audit**: Violations are logged but pods are still created.
- **warn**: Violations generate warnings but pods are still created.

## Step 1: Check Current PSA Configuration

Verify whether your cluster has PSA enabled:

```bash
# Check the Kubernetes version (PSA is stable in 1.25+)
kubectl version --short

# Check namespace labels for existing PSA configuration
kubectl get namespaces -o json | \
  jq -r '.items[] | select(.metadata.labels["pod-security.kubernetes.io/enforce"] != null) | "\(.metadata.name): enforce=\(.metadata.labels["pod-security.kubernetes.io/enforce"])"'
```

## Step 2: Configure PSA at the Namespace Level

Apply Pod Security Standards to namespaces using labels:

```bash
# Apply Baseline enforcement to a namespace
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/enforce-version=latest \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/warn=restricted

# Apply Restricted enforcement to a sensitive namespace
kubectl label namespace sensitive-data \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=latest \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/warn=restricted
```

This configuration enforces baseline security, audits against restricted standards, and warns about restricted violations.

## Step 3: Configure PSA Through Rancher UI

Rancher provides a UI for managing PSA on projects and namespaces:

1. Navigate to your cluster in Rancher.
2. Go to **Cluster > Projects/Namespaces**.
3. Click on the namespace you want to configure.
4. Under **Labels**, add the PSA labels:

```plaintext
pod-security.kubernetes.io/enforce = baseline
pod-security.kubernetes.io/enforce-version = latest
pod-security.kubernetes.io/audit = restricted
pod-security.kubernetes.io/warn = restricted
```

5. Click **Save**.

## Step 4: Configure PSA at the Cluster Level

To set a default PSA level for the entire cluster, configure the admission controller:

Create an admission configuration file:

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
  - name: PodSecurity
    configuration:
      apiVersion: pod-security.admission.config.k8s.io/v1
      kind: PodSecurityConfiguration
      defaults:
        enforce: "baseline"
        enforce-version: "latest"
        audit: "restricted"
        audit-version: "latest"
        warn: "restricted"
        warn-version: "latest"
      exemptions:
        usernames: []
        runtimeClasses: []
        namespaces:
          - kube-system
          - cattle-system
          - cattle-fleet-system
          - cattle-impersonation-system
          - rancher-operator-system
```

For RKE2 clusters managed by Rancher, place this in the cluster configuration:

1. Go to the cluster's settings in Rancher.
2. Edit the cluster YAML.
3. Add the admission configuration under `spec.rkeConfig.machineGlobalConfig`:

```yaml
spec:
  rkeConfig:
    machineGlobalConfig:
      kube-apiserver-arg:
        - "admission-control-config-file=/etc/rancher/rke2/psa.yaml"
```

## Step 5: Set Up PSA for Rancher Projects

Apply PSA labels to all namespaces in a Rancher project consistently:

```bash
#!/bin/bash
# apply-psa-to-project.sh

PROJECT_NAMESPACE="c-m-xxxxx"
PROJECT_ID="p-xxxxx"
PSA_LEVEL="baseline"

# Get all namespaces in the project
NAMESPACES=$(kubectl get namespaces -o json | \
  jq -r ".items[] | select(.metadata.annotations[\"field.cattle.io/projectId\"] == \"$PROJECT_NAMESPACE:$PROJECT_ID\") | .metadata.name")

for ns in $NAMESPACES; do
  echo "Applying PSA labels to namespace: $ns"
  kubectl label namespace $ns \
    pod-security.kubernetes.io/enforce=$PSA_LEVEL \
    pod-security.kubernetes.io/enforce-version=latest \
    pod-security.kubernetes.io/audit=restricted \
    pod-security.kubernetes.io/warn=restricted \
    --overwrite
done
```

## Step 6: Test PSA Enforcement

Test that PSA is working by trying to create a pod that violates the policy:

**Test against Baseline enforcement:**

```yaml
# This pod should be rejected under baseline enforcement
apiVersion: v1
kind: Pod
metadata:
  name: test-privileged
  namespace: production
spec:
  containers:
    - name: test
      image: nginx
      securityContext:
        privileged: true
```

```bash
kubectl apply -f test-privileged.yaml
# Expected: Error - pod violates the baseline Pod Security Standard
```

**Test a compliant pod:**

```yaml
# This pod should be accepted under baseline enforcement
apiVersion: v1
kind: Pod
metadata:
  name: test-compliant
  namespace: production
spec:
  containers:
    - name: test
      image: nginx
      securityContext:
        allowPrivilegeEscalation: false
        runAsNonRoot: true
        runAsUser: 1000
        seccompProfile:
          type: RuntimeDefault
        capabilities:
          drop:
            - ALL
```

```bash
kubectl apply -f test-compliant.yaml
# Expected: Pod created successfully
```

## Step 7: Exempt System Namespaces

Rancher and Kubernetes system namespaces need to run privileged workloads. Exempt them:

```bash
# Exempt Rancher system namespaces
for ns in kube-system cattle-system cattle-fleet-system cattle-impersonation-system; do
  kubectl label namespace $ns \
    pod-security.kubernetes.io/enforce=privileged \
    pod-security.kubernetes.io/audit=privileged \
    pod-security.kubernetes.io/warn=privileged \
    --overwrite
done
```

## Step 8: Roll Out PSA Gradually

Use a phased approach to avoid disrupting existing workloads:

**Phase 1 - Audit only:**

```bash
kubectl label namespace production \
  pod-security.kubernetes.io/audit=baseline \
  pod-security.kubernetes.io/warn=baseline
```

Review the audit logs and warnings:

```bash
# Check for PSA violations in the audit log
kubectl get events -n production --field-selector reason=FailedCreate | grep "violates PodSecurity"

# Check warnings in recent pod creations
kubectl logs -n kube-system -l component=kube-apiserver | grep "PodSecurity"
```

**Phase 2 - Warn and audit with restricted, enforce baseline:**

```bash
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/warn=restricted \
  --overwrite
```

**Phase 3 - Enforce restricted:**

```bash
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=restricted \
  --overwrite
```

## Step 9: Monitor PSA Violations

Set up monitoring for PSA violations:

```bash
# Check for audit annotations on pods
kubectl get pods -n production -o json | \
  jq -r '.items[] | select(.metadata.annotations["pod-security.kubernetes.io/audit-violations"] != null) | "\(.metadata.name): \(.metadata.annotations["pod-security.kubernetes.io/audit-violations"])"'
```

## Step 10: Document Exemption Procedures

Create a process for handling workloads that need elevated privileges:

1. The team submits a request explaining why elevated privileges are needed.
2. The platform team reviews the request and the pod's security context.
3. If approved, the workload is placed in a namespace with appropriate PSA labels.
4. The exemption is documented and reviewed periodically.

```yaml
# Create a dedicated namespace for privileged workloads
apiVersion: v1
kind: Namespace
metadata:
  name: privileged-workloads
  labels:
    pod-security.kubernetes.io/enforce: privileged
    pod-security.kubernetes.io/audit: baseline
    pod-security.kubernetes.io/warn: baseline
  annotations:
    field.cattle.io/projectId: "c-m-xxxxx:p-xxxxx"
    psa-exemption-reason: "Contains system monitoring agents requiring host access"
    psa-exemption-approved-by: "platform-team"
    psa-exemption-review-date: "2026-06-19"
```

## Best Practices

- **Start with audit mode**: Begin by auditing in restricted mode to identify violations before enforcing anything.
- **Exempt system namespaces**: Always exempt Rancher and Kubernetes system namespaces.
- **Use baseline as the minimum**: Enforce at least the baseline standard for all user-facing namespaces.
- **Target restricted for production**: Work toward restricted enforcement for production workloads.
- **Phase the rollout**: Move gradually from audit to warn to enforce.
- **Document exemptions**: Maintain records of any namespace that runs with relaxed security standards.
- **Apply consistently**: Use scripts or automation to apply PSA labels consistently across all namespaces in a project.

## Conclusion

Pod Security Admission in Rancher provides a built-in mechanism for enforcing pod security standards across your clusters. By configuring PSA at the namespace and cluster level, exempting system namespaces, and rolling out enforcement gradually, you can harden your workloads without disrupting existing applications. Start with audit mode, fix violations, and progressively tighten enforcement to reach the restricted standard for production workloads.
