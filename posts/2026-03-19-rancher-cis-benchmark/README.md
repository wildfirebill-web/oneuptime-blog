# How to Run CIS Benchmark Scans in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Security, CIS Benchmark

Description: Learn how to run CIS Kubernetes Benchmark scans in Rancher to assess and improve the security posture of your clusters.

The CIS (Center for Internet Security) Kubernetes Benchmark provides a set of security recommendations for hardening Kubernetes clusters. Rancher includes a built-in CIS scanning feature that evaluates your clusters against these benchmarks and reports on compliance. This guide shows you how to run scans and remediate findings.

## Prerequisites

- Rancher v2.5 or later
- Admin access to Rancher
- RKE, RKE2, or K3s managed clusters
- The Rancher CIS Benchmark application installed

## Step 1: Install the CIS Benchmark Application

### Via the Rancher UI

1. Navigate to the downstream cluster where you want to run scans.
2. Go to **Apps & Marketplace** > **Charts**.
3. Search for **CIS Benchmark**.
4. Click **Install**.
5. Accept the default settings or customize the namespace.
6. Click **Install** to deploy.

### Via Helm

```bash
helm repo add rancher-charts https://charts.rancher.io
helm repo update

helm install rancher-cis-benchmark rancher-charts/rancher-cis-benchmark \
  -n cis-operator-system \
  --create-namespace
```

Verify the installation:

```bash
kubectl get pods -n cis-operator-system
```

## Step 2: Run a CIS Benchmark Scan

### Via the Rancher UI

1. Navigate to the cluster.
2. Go to **CIS Benchmark** in the left navigation.
3. Click **Run Scan**.
4. Select the benchmark profile:
   - **CIS 1.6** for general Kubernetes
   - **CIS 1.6 Hardened** for RKE2 hardened clusters
   - **RKE-CIS-1.6** for RKE-specific benchmarks
5. Click **Run**.

### Via kubectl

Create a ClusterScan resource:

```yaml
apiVersion: cis.cattle.io/v1
kind: ClusterScan
metadata:
  name: cis-scan-1
spec:
  scanProfileName: cis-1.6-profile
```

Apply it:

```bash
kubectl apply -f cis-scan.yaml
```

## Step 3: Monitor Scan Progress

Watch the scan status:

```bash
kubectl get clusterscan cis-scan-1 -o yaml
```

The scan typically takes 5-15 minutes depending on cluster size. You can also watch progress in the Rancher UI under the CIS Benchmark section.

## Step 4: Review Scan Results

### In the Rancher UI

1. Go to **CIS Benchmark** > **Scans**.
2. Click on the completed scan.
3. Review results categorized as:
   - **Pass**: The check passed.
   - **Fail**: The check failed and needs remediation.
   - **Skip**: The check was skipped (not applicable or manually exempted).
   - **Not Applicable**: The check does not apply to this cluster type.

### Via kubectl

```bash
kubectl get clusterscanreports -o yaml
```

Get a summary:

```bash
kubectl get clusterscan cis-scan-1 -o jsonpath='{.status.summary}'
```

## Step 5: Understand Common Failures

Here are frequently failed checks and how to remediate them:

### 1.2.6 - Ensure that the --kubelet-certificate-authority argument is set

Add the kubelet certificate authority to the API server configuration:

```yaml
# RKE2 config.yaml

kube-apiserver-arg:
  - "kubelet-certificate-authority=/var/lib/rancher/rke2/server/tls/server-ca.crt"
```

### 4.2.6 - Ensure that the --protect-kernel-defaults argument is set to true

On each node, configure the kubelet:

```yaml
# RKE2 config.yaml
kubelet-arg:
  - "protect-kernel-defaults=true"
```

Set the required kernel parameters first:

```bash
cat >> /etc/sysctl.d/90-kubelet.conf << 'EOF'
vm.overcommit_memory=1
vm.panic_on_oom=0
kernel.panic=10
kernel.panic_on_oops=1
kernel.keys.root_maxbytes=25000000
EOF

sysctl --system
```

### 5.2.2 - Minimize the admission of containers with allowPrivilegeEscalation

Create a Pod Security Policy or Pod Security Standard that restricts privilege escalation:

```yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: restricted
spec:
  allowPrivilegeEscalation: false
  requiredDropCapabilities:
  - ALL
  runAsUser:
    rule: MustRunAsNonRoot
```

## Step 6: Schedule Recurring Scans

Create a scheduled scan to run weekly:

```yaml
apiVersion: cis.cattle.io/v1
kind: ClusterScan
metadata:
  name: weekly-cis-scan
spec:
  scanProfileName: cis-1.6-profile
  scheduledScanConfig:
    cronSchedule: "0 6 * * 1"
    retentionCount: 10
```

Apply it:

```bash
kubectl apply -f weekly-scan.yaml
```

## Step 7: Create Custom Scan Profiles

If certain checks are not applicable to your environment, create a custom profile that skips them:

```yaml
apiVersion: cis.cattle.io/v1
kind: ClusterScanProfile
metadata:
  name: custom-cis-profile
spec:
  benchmarkVersion: cis-1.6
  skipTests:
  - "1.2.6"
  - "1.2.16"
  - "4.2.6"
```

Apply the custom profile:

```bash
kubectl apply -f custom-profile.yaml
```

Use it in a scan:

```yaml
apiVersion: cis.cattle.io/v1
kind: ClusterScan
metadata:
  name: custom-scan
spec:
  scanProfileName: custom-cis-profile
```

## Step 8: Export Scan Reports

Export scan results for compliance documentation:

### Via the Rancher UI

1. Go to the completed scan.
2. Click **Download Report** to get a CSV or PDF export.

### Via kubectl

```bash
kubectl get clusterscanreports -o json > cis-report.json
```

## Step 9: Set Up Alerting on Scan Failures

Create alerts for failed CIS scans:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: cis-scan-alerts
  namespace: cattle-monitoring-system
spec:
  groups:
  - name: cis-benchmark
    rules:
    - alert: CISScanFailed
      expr: |
        cis_scan_fail_count > 0
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "CIS benchmark scan has failing checks"
```

## Best Practices

- Run CIS scans before deploying a cluster to production.
- Schedule weekly or monthly recurring scans.
- Create custom profiles to skip checks that are genuinely not applicable, but document why each is skipped.
- Track compliance improvement over time by comparing scan results.
- Integrate scan results into your compliance reporting workflow.
- Remediate critical failures immediately and plan fixes for warnings.

## Conclusion

CIS benchmark scanning in Rancher provides visibility into the security posture of your Kubernetes clusters. By running regular scans, understanding the results, and systematically remediating failures, you can maintain compliance with industry security standards and reduce the risk of security incidents.
