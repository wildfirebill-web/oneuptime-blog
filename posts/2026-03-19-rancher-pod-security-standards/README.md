# How to Configure Pod Security Standards in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Security, Pod Security

Description: Learn how to configure Pod Security Standards in Rancher to enforce security baselines for pods using the built-in admission controller.

Pod Security Standards (PSS) replace the deprecated Pod Security Policies in Kubernetes 1.25+. They provide three predefined security levels enforced through the Pod Security Admission (PSA) controller. Rancher supports PSS configuration through namespace labels and cluster-wide defaults. This guide covers setting up PSS in your Rancher-managed clusters.

## Prerequisites

- Rancher v2.7 or later
- Kubernetes 1.23+ clusters (PSA is stable in 1.25+)
- Admin access to Rancher
- kubectl access to the cluster

## Step 1: Understand Pod Security Standards Levels

PSS defines three security levels:

**Privileged**: Unrestricted policy. Allows all pod configurations. Use only for system-level workloads that truly need elevated access.

**Baseline**: Minimally restrictive policy. Prevents known privilege escalations while remaining compatible with most workloads. Blocks hostNetwork, hostPID, privileged containers, and most host path mounts.

**Restricted**: Heavily restricted policy. Follows security best practices. Requires non-root users, drops all capabilities, and enforces read-only root filesystems.

Each level can be applied in three modes:

- **enforce**: Reject pods that violate the policy.
- **audit**: Log violations but allow the pod.
- **warn**: Display a warning to the user but allow the pod.

## Step 2: Configure PSS at the Namespace Level

Apply PSS to a namespace using labels:

```bash
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/warn=restricted
```

For a less restrictive namespace:

```bash
kubectl label namespace staging \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/warn=restricted
```

This enforces baseline but warns on restricted violations, helping you prepare for tighter security.

## Step 3: Configure PSS via Rancher UI

1. Navigate to the downstream cluster in Rancher.
2. Go to **Cluster** > **Projects/Namespaces**.
3. Click on a namespace.
4. Under **Pod Security Admission**, select the desired level and mode.
5. Save the changes.

For new namespaces:

1. Click **Create Namespace**.
2. In the creation form, set the Pod Security labels.
3. Create the namespace.

## Step 4: Set Cluster-Wide Defaults

Configure default PSS for all new namespaces by setting up an AdmissionConfiguration. Create the configuration file:

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
- name: PodSecurity
  configuration:
    apiVersion: pod-security.admission.config.k8s.io/v1
    kind: PodSecurityConfiguration
    defaults:
      enforce: baseline
      enforce-version: latest
      audit: restricted
      audit-version: latest
      warn: restricted
      warn-version: latest
    exemptions:
      usernames: []
      runtimeClasses: []
      namespaces:
      - kube-system
      - cattle-system
      - cattle-fleet-system
      - cattle-impersonation-system
      - cis-operator-system
      - cattle-resources-system
```

For RKE2 clusters, place this file on the server node:

```bash
mkdir -p /etc/rancher/rke2/
cat > /etc/rancher/rke2/rancher-pss.yaml << 'EOF'
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
- name: PodSecurity
  configuration:
    apiVersion: pod-security.admission.config.k8s.io/v1
    kind: PodSecurityConfiguration
    defaults:
      enforce: baseline
      enforce-version: latest
      audit: restricted
      audit-version: latest
      warn: restricted
      warn-version: latest
    exemptions:
      namespaces:
      - kube-system
      - cattle-system
      - cattle-fleet-system
EOF
```

Reference it in the RKE2 config:

```yaml
# /etc/rancher/rke2/config.yaml

kube-apiserver-arg:
  - "admission-control-config-file=/etc/rancher/rke2/rancher-pss.yaml"
```

## Step 5: Exempt System Namespaces

System namespaces typically need privileged access. Exempt them in the cluster-wide configuration (as shown above) or by labeling them explicitly:

```bash
kubectl label namespace kube-system \
  pod-security.kubernetes.io/enforce=privileged

kubectl label namespace cattle-system \
  pod-security.kubernetes.io/enforce=privileged

kubectl label namespace cattle-fleet-system \
  pod-security.kubernetes.io/enforce=privileged
```

## Step 6: Test PSS Enforcement

Deploy a pod that violates the restricted policy:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-violation
  namespace: production
spec:
  containers:
  - name: test
    image: nginx
    securityContext:
      privileged: true
```

```bash
kubectl apply -f test-violation.yaml
```

Expected output when enforce is set to restricted:

```plaintext
Error from server (Forbidden): error when creating "test-violation.yaml":
pods "test-violation" is forbidden: violates PodSecurity "restricted:latest":
privileged (container "test" must not set securityContext.privileged=true)
```

Deploy a compliant pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-compliant
  namespace: production
spec:
  securityContext:
    runAsNonRoot: true
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: test
    image: nginx
    securityContext:
      runAsUser: 1000
      allowPrivilegeEscalation: false
      capabilities:
        drop:
        - ALL
```

## Step 7: Gradual Rollout Strategy

Roll out PSS gradually to avoid breaking existing workloads:

1. **Phase 1 - Audit Only**: Apply restricted in audit mode to all namespaces:

```bash
kubectl label namespace --all \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/warn=restricted
```

2. **Phase 2 - Review Violations**: Check audit logs for violations:

```bash
kubectl logs -n kube-system -l component=kube-apiserver | grep "pod-security"
```

3. **Phase 3 - Fix Workloads**: Update workload manifests to comply with the target level.

4. **Phase 4 - Enforce**: Switch from audit to enforce mode:

```bash
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=restricted \
  --overwrite
```

## Step 8: Monitor PSS Violations

Set up monitoring for PSS audit events:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: pss-alerts
  namespace: cattle-monitoring-system
spec:
  groups:
  - name: pod-security
    rules:
    - alert: PodSecurityViolation
      expr: |
        increase(apiserver_admission_webhook_rejection_count{
          name="pod-security"
        }[5m]) > 0
      labels:
        severity: warning
      annotations:
        summary: "Pod Security Standard violations detected"
```

## Conclusion

Pod Security Standards provide a straightforward way to enforce security baselines across your Rancher-managed clusters. By using namespace labels with enforce, audit, and warn modes, you can gradually roll out security restrictions without disrupting existing workloads. Start with audit mode, fix violations, and progressively move to enforcement for a smooth transition.
