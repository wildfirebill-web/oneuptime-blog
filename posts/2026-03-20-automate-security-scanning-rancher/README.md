# How to Automate Security Scanning in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Security Scanning, Trivy, Falco, CIS, Kubernetes, Vulnerability Management

Description: Automate security scanning in Rancher using Trivy for container image vulnerabilities, CIS benchmark scanning for cluster hardening, NeuVector for runtime scanning, and integrate findings into CI/CD pipelines.

## Introduction

Security scanning in Rancher must be continuous—new CVEs emerge daily, configurations drift, and new images are deployed constantly. Automating security scanning at multiple layers (images in CI/CD, running containers, cluster configuration) creates a continuous security posture that catches vulnerabilities before they become incidents.

## Step 1: Image Scanning with Trivy Operator

```bash
# Install Trivy Operator for continuous image scanning
helm repo add aqua https://aquasecurity.github.io/helm-charts/
helm install trivy-operator aqua/trivy-operator \
  --namespace trivy-system \
  --create-namespace \
  --set trivy.ignoreUnfixed=true \
  --set operator.scanJobTimeout=5m \
  --set operator.concurrentScanJobsLimit=10 \
  --set compliance.failEntriesLimit=10
```

```yaml
# Configure scan job template
apiVersion: v1
kind: ConfigMap
metadata:
  name: trivy-operator
  namespace: trivy-system
data:
  scanJob.tolerations: '[{"operator":"Exists"}]'
  # Severity thresholds
  vulnerabilityReports.scanner.severity: "MEDIUM,HIGH,CRITICAL"
  # Compliance standards
  compliance.failEntriesLimit: "10"
```

## Step 2: Automated CIS Benchmark Scanning

```yaml
# Schedule weekly CIS benchmark scans
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cis-benchmark-weekly
  namespace: cattle-system
spec:
  schedule: "0 6 * * 0"    # Every Sunday at 6 AM
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: cis-scanner-sa
          containers:
            - name: cis-scanner
              image: myregistry/rancher-cis-scanner:latest
              command:
                - /bin/sh
                - -c
                - |
                  kubectl apply -f - <<EOF
                  apiVersion: cis.cattle.io/v1
                  kind: ClusterScan
                  metadata:
                    name: weekly-cis-$(date +%Y%m%d)
                  spec:
                    scanProfileName: rke2-cis-1.24-profile
                  EOF
                  # Wait for scan to complete
                  kubectl wait --for=condition=complete \
                    clusterscan/weekly-cis-$(date +%Y%m%d) \
                    --timeout=30m
                  # Export results
                  kubectl get clusterscansummary weekly-cis-$(date +%Y%m%d) \
                    -o json > /results/cis-scan-$(date +%Y%m%d).json
          restartPolicy: OnFailure
```

## Step 3: CI/CD Pipeline Image Scanning

```yaml
# GitHub Actions: scan images before push to registry
name: Container Security Scan
on:
  push:
    paths:
      - 'Dockerfile*'

jobs:
  scan-image:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build image
        run: docker build -t myapp:${{ github.sha }} .

      - name: Run Trivy scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: myapp:${{ github.sha }}
          format: 'table'
          exit-code: '1'              # Fail build on CRITICAL
          ignore-unfixed: true
          severity: 'CRITICAL,HIGH'

      - name: Upload SARIF results
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: trivy-results.sarif
```

## Step 4: Runtime Security with Falco

```bash
# Install Falco for runtime threat detection
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm install falco falcosecurity/falco \
  --namespace falco-system \
  --create-namespace \
  --set tty=true \
  --set falco.grpc.enabled=true \
  --set falcoctl.artifact.install.enabled=true
```

```yaml
# Custom Falco rules for Rancher environment
apiVersion: v1
kind: ConfigMap
metadata:
  name: falco-custom-rules
  namespace: falco-system
data:
  custom_rules.yaml: |
    # Alert on privilege escalation
    - rule: Privilege Escalation via sudo
      desc: Detect sudo usage in containers
      condition: spawned_process and proc.name = sudo and container
      output: "Privilege escalation in container (user=%user.name cmd=%proc.cmdline)"
      priority: WARNING

    # Alert on sensitive file reads
    - rule: Read Sensitive Files
      desc: Detect reading of certificates and keys
      condition: >
        open_read and fd.name startswith /etc/ssl and
        not proc.name in (nginx, apache2, envoy) and container
      output: "Sensitive file read in container (file=%fd.name user=%user.name)"
      priority: WARNING

    # Alert on network scanning
    - rule: Network Port Scanning
      desc: Detect port scanning from containers
      condition: outbound and evt.type = connect and fd.sport < 1024 and container
      output: "Port scanning detected from container"
      priority: CRITICAL
```

## Step 5: Vulnerability Reports Dashboard

```yaml
# Grafana dashboard for vulnerability metrics
# Using Trivy Operator Prometheus metrics

# PrometheusRule for vulnerability alerts
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: vulnerability-alerts
  namespace: trivy-system
spec:
  groups:
    - name: vulnerabilities
      rules:
        - alert: CriticalVulnerabilityDetected
          expr: |
            sum(trivy_image_vulnerabilities{severity="CRITICAL"}) > 0
          for: 0m
          annotations:
            summary: "Critical vulnerability detected in deployed images"
          labels:
            severity: critical

        - alert: HighVulnerabilityCount
          expr: |
            sum(trivy_image_vulnerabilities{severity="HIGH"}) > 20
          for: 1h
          annotations:
            summary: "High vulnerability count exceeds threshold"
          labels:
            severity: warning
```

## Step 6: Automated Remediation

```bash
#!/bin/bash
# auto_remediate.sh - Quarantine pods with critical vulnerabilities

# Get pods with critical vulnerabilities
CRITICAL_PODS=$(kubectl get vulnerabilityreports -A -o json | \
  jq -r '.items[] | select(.report.summary.criticalCount > 0) |
    "\(.metadata.namespace)/\(.metadata.labels["trivy-operator.pod.name"])"')

for pod_ref in $CRITICAL_PODS; do
  NAMESPACE=$(echo $pod_ref | cut -d'/' -f1)
  POD=$(echo $pod_ref | cut -d'/' -f2)

  echo "CRITICAL vulnerability in pod $POD (namespace: $NAMESPACE)"

  # Add label to quarantine (prevents traffic routing)
  kubectl label pod "$POD" -n "$NAMESPACE" \
    security-quarantine=true \
    --overwrite

  # Notify security team
  curl -X POST "$SLACK_WEBHOOK" \
    -d "{\"text\": \"Security quarantine: pod $POD in $NAMESPACE has CRITICAL vulnerabilities\"}"
done
```

## Conclusion

Automated security scanning in Rancher creates continuous visibility into vulnerabilities across the entire Kubernetes environment. Trivy Operator scans running containers continuously, CIS benchmark scans run weekly to catch configuration drift, and CI/CD pipeline scanning blocks critical vulnerabilities before deployment. Falco provides runtime threat detection for active attack indicators. Feed all findings into a central dashboard and alert on critical issues immediately—automated remediation (quarantine) for pods with critical CVEs reduces exposure time.
