# How to Automate Security Scanning in Rancher - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Security-scanning, CIS, NeuVector, Trivy, Automation

Description: A guide to automating container image scanning, CIS benchmark scanning, and runtime security monitoring in Rancher environments.

## Overview

Security scanning in Rancher environments encompasses multiple dimensions: container image vulnerability scanning in CI/CD pipelines, CIS benchmark compliance scanning for cluster configurations, and runtime security monitoring for threats. Automating these scans provides continuous security assurance and helps teams catch vulnerabilities before they reach production. This guide covers setting up automated security scanning across all layers.

## Level 1: Image Vulnerability Scanning

### Trivy in CI/CD Pipelines

```yaml
# GitHub Actions: Scan images on every push

name: Security Scan
on: [push, pull_request]

jobs:
  image-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build Docker image
        run: docker build -t myapp:${{ github.sha }} .

      - name: Run Trivy vulnerability scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: 'myapp:${{ github.sha }}'
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'
          exit-code: '1'    # Fail build on critical/high CVEs

      - name: Upload Trivy scan results to GitHub Security
        uses: github/codeql-action/upload-sarif@v2
        if: always()
        with:
          sarif_file: 'trivy-results.sarif'
```

### NeuVector Image Scanning in CI/CD

```bash
#!/bin/bash
# neuvector-scan.sh - Scan image with NeuVector
NEUVECTOR_URL="${NEUVECTOR_URL}"
IMAGE="$1"
TAG="$2"

# Get auth token
TOKEN=$(curl -s -k \
  -X POST "${NEUVECTOR_URL}/auth" \
  -H "Content-Type: application/json" \
  -d '{"password": {"username": "admin", "password": "${NEUVECTOR_PASS}"}}' \
  | jq -r '.token.token')

# Submit scan request
SCAN_ID=$(curl -s -k \
  -X POST "${NEUVECTOR_URL}/v1/scan/image" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d "{
    \"request\": {
      \"registry\": \"https://registry.example.com\",
      \"repository\": \"${IMAGE}\",
      \"tag\": \"${TAG}\"
    }
  }" | jq -r '.token')

# Poll for results
for i in $(seq 1 30); do
  RESULT=$(curl -s -k \
    -H "Authorization: Bearer ${TOKEN}" \
    "${NEUVECTOR_URL}/v1/scan/image/${SCAN_ID}")

  STATUS=$(echo "${RESULT}" | jq -r '.report.status')

  if [ "${STATUS}" = "finished" ]; then
    # Count critical and high CVEs
    CRITICAL=$(echo "${RESULT}" | jq '[.report.vuls[] | select(.severity == "Critical")] | length')
    HIGH=$(echo "${RESULT}" | jq '[.report.vuls[] | select(.severity == "High")] | length')

    echo "Scan complete: Critical=${CRITICAL}, High=${HIGH}"

    if [ "${CRITICAL}" -gt 0 ] || [ "${HIGH}" -gt 5 ]; then
      echo "FAIL: Too many vulnerabilities (Critical: ${CRITICAL}, High: ${HIGH})"
      exit 1
    fi
    exit 0
  fi

  sleep 10
done

echo "FAIL: Scan timed out"
exit 1
```

## Level 2: CIS Benchmark Scanning

### Automated CIS Scans with Rancher

```yaml
# Schedule weekly CIS benchmark scans
apiVersion: cis.cattle.io/v1
kind: ClusterScanBenchmark
metadata:
  name: rke2-cis-hardened
  namespace: cis-operator-system
spec:
  clusterProvider: rke2
  minKubernetesVersion: "1.24"
  customBenchmarkConfigMapName: rke2-cis-1.23
---
# Run scan weekly
apiVersion: cis.cattle.io/v1
kind: ScheduledClusterScan
metadata:
  name: weekly-cis-scan
  namespace: cis-operator-system
spec:
  enabled: true
  cronSchedule: "0 2 * * 0"   # Sunday 2 AM
  scanConfig:
    scanProfileName: rke2-cis-1.23-hardened
  retentionCount: 12           # Keep 12 weeks of scans
```

### Export CIS Scan Results

```bash
#!/bin/bash
# export-cis-report.sh - Export and send CIS scan results

CLUSTER_ID="$1"

# Get the latest scan report
SCAN_REPORT=$(kubectl get clusterscans -n cis-operator-system \
  -o jsonpath='{.items[-1:].status.summary}')

PASS=$(echo "${SCAN_REPORT}" | jq -r '.pass')
FAIL=$(echo "${SCAN_REPORT}" | jq -r '.fail')
WARN=$(echo "${SCAN_REPORT}" | jq -r '.warn')
TOTAL=$(echo "${SCAN_REPORT}" | jq -r '.total')

echo "CIS Scan Results: Pass=${PASS}, Fail=${FAIL}, Warn=${WARN}, Total=${TOTAL}"

# Calculate pass percentage
PASS_PCT=$(echo "scale=1; ${PASS} * 100 / ${TOTAL}" | bc)

# Alert if pass rate is below 90%
if (( $(echo "${PASS_PCT} < 90" | bc -l) )); then
  curl -X POST "${SLACK_WEBHOOK}" \
    -d "{\"text\": \":warning: CIS Scan: ${PASS_PCT}% pass rate (${FAIL} failures)\"}"
fi

# Export full report as PDF (via Rancher UI API)
curl -s -k \
  -H "Authorization: Bearer ${RANCHER_TOKEN}" \
  "${RANCHER_URL}/v3/clusterscans/${SCAN_NAME}?action=download" \
  -o "cis-report-$(date +%Y%m%d).pdf"
```

## Level 3: Runtime Security with NeuVector

### Automated Policy Learning and Enforcement

```bash
#!/bin/bash
# neuvector-enforce-mode.sh
# Move groups from Learning to Monitor mode after 7 days

NEUVECTOR_URL="${NEUVECTOR_URL}"
DAYS_THRESHOLD=7

TOKEN=$(curl -s -k \
  -X POST "${NEUVECTOR_URL}/auth" \
  -d '{"password": {"username": "admin", "password": "admin"}}' \
  | jq -r '.token.token')

# Get all groups in Learning mode
GROUPS=$(curl -s -k \
  -H "Authorization: Bearer ${TOKEN}" \
  "${NEUVECTOR_URL}/v1/group" \
  | jq -r '.groups[] | select(.profile_mode == "discover") | .name')

NOW=$(date +%s)

for GROUP in ${GROUPS}; do
  # Get group creation time
  CREATED=$(curl -s -k \
    -H "Authorization: Bearer ${TOKEN}" \
    "${NEUVECTOR_URL}/v1/group/${GROUP}" \
    | jq -r '.group.created_timestamp')

  DAYS_OLD=$(( (NOW - CREATED) / 86400 ))

  if [ "${DAYS_OLD}" -ge "${DAYS_THRESHOLD}" ]; then
    echo "Moving ${GROUP} from discover to monitor mode (${DAYS_OLD} days old)"

    # Switch to monitor mode
    curl -s -k \
      -X PATCH "${NEUVECTOR_URL}/v1/group/${GROUP}" \
      -H "Authorization: Bearer ${TOKEN}" \
      -H "Content-Type: application/json" \
      -d '{"config": {"profile_mode": "monitor"}}' \
      > /dev/null
  fi
done
```

## Level 4: Kubernetes Audit Log Analysis

```yaml
# Deploy Falco for audit log anomaly detection
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: falco
  namespace: falco-system
spec:
  selector:
    matchLabels:
      app: falco
  template:
    spec:
      hostPID: true
      hostNetwork: true
      containers:
        - name: falco
          image: falcosecurity/falco-no-driver:latest
          securityContext:
            privileged: true
          args:
            - /usr/bin/falco
            - --cri
            - /run/containerd/containerd.sock
            - -K
            - /var/run/secrets/kubernetes.io/serviceaccount/token
            - -k
            - https://$(KUBERNETES_SERVICE_HOST)
            - --k8s-api-cert
            - /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
          volumeMounts:
            - mountPath: /host/var/run/docker.sock
              name: docker-socket
            - mountPath: /run/containerd/containerd.sock
              name: containerd-socket
```

## Aggregated Security Dashboard

```yaml
# PrometheusRule to track security scan metrics
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: security-scanning-alerts
  namespace: cattle-monitoring-system
spec:
  groups:
    - name: security-scanning
      rules:
        - alert: CriticalCVEFound
          expr: neuvector_image_critical_vulnerabilities > 0
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "Critical CVE found in running container image"
        - alert: CISScoreDropped
          expr: cis_benchmark_pass_percentage < 90
          for: 1h
          labels:
            severity: warning
          annotations:
            summary: "CIS benchmark score dropped below 90%"
```

## Conclusion

Automating security scanning in Rancher requires a multi-layered approach: image scanning in CI/CD pipelines catches vulnerabilities before deployment, CIS benchmark scanning ensures cluster configurations meet compliance standards, and runtime security monitoring with NeuVector and Falco detects threats in production. Schedule regular automated scans, integrate results into your monitoring dashboards, and set up alerts for critical findings. Security scanning should be continuous, not a one-time activity.
