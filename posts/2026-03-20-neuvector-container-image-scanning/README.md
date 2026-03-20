# How to Run Container Image Scanning with NeuVector

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: NeuVector, Container Security, Vulnerability Scanning, CVE, Kubernetes

Description: A comprehensive guide to running container image vulnerability scans with NeuVector, covering on-demand scanning, automated scanning, and result interpretation.

## Introduction

Container image scanning is one of NeuVector's core capabilities. It detects known CVEs (Common Vulnerabilities and Exposures), embedded secrets, and compliance issues in container images before and after deployment. This guide covers how to set up and run image scans using the NeuVector UI, REST API, and command-line tools.

## Prerequisites

- NeuVector installed with Scanner component
- Images in a accessible registry or running in the cluster
- NeuVector Manager access

## Understanding Scan Types

NeuVector supports several scanning approaches:

| Scan Type | When It Runs | Use Case |
|---|---|---|
| On-demand | Manually triggered | Ad hoc image checks |
| Auto-scan | On new image detection | Continuous monitoring |
| Registry scan | Scheduled or on-demand | Full registry audit |
| Admission control | Before pod creation | Gate deployments |

## Step 1: Scan a Running Container

Scan containers already running in your cluster:

1. Navigate to **Assets** > **Containers** in the NeuVector UI
2. Find the container you want to scan
3. Click the container name to open details
4. Click **Scan** to trigger an immediate vulnerability scan

Via REST API:

```bash
# Get auth token

TOKEN=$(curl -sk -X POST \
  "https://neuvector-manager:8443/v1/auth" \
  -H "Content-Type: application/json" \
  -d '{"password":{"username":"admin","password":"yourpassword"}}' \
  | jq -r '.token.token')

# List running workloads to get the container ID
curl -sk "https://neuvector-manager:8443/v1/workload?brief=true" \
  -H "X-Auth-Token: ${TOKEN}" | jq '.workloads[].id'

# Trigger a scan on a specific container
CONTAINER_ID="abc123def456"
curl -sk -X POST \
  "https://neuvector-manager:8443/v1/scan/workload/${CONTAINER_ID}" \
  -H "X-Auth-Token: ${TOKEN}"
```

## Step 2: Scan an Image by Tag

Scan a specific image before deploying it:

```bash
# Scan an image directly
curl -sk -X POST \
  "https://neuvector-manager:8443/v1/scan/image" \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${TOKEN}" \
  -d '{
    "request": {
      "tag": "nginx:1.24",
      "registry": "https://registry-1.docker.io",
      "username": "",
      "password": ""
    }
  }'

# Get scan results for the image
curl -sk \
  "https://neuvector-manager:8443/v1/scan/image/nginx%3A1.24" \
  -H "X-Auth-Token: ${TOKEN}" | jq '.report'
```

## Step 3: Enable Auto-Scan for New Images

Configure NeuVector to automatically scan images when new containers start:

```bash
# Enable auto-scan via API
curl -sk -X PATCH \
  "https://neuvector-manager:8443/v1/scan/config" \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${TOKEN}" \
  -d '{
    "config": {
      "auto_scan": true,
      "scan_layers": true
    }
  }'
```

In the UI:
1. Go to **Configuration** > **Scanning**
2. Enable **Auto-Scan** toggle
3. Enable **Scan Layers** for detailed layer-by-layer analysis

## Step 4: Interpret Scan Results

Understanding the scan report output:

```bash
# Get detailed scan results
curl -sk \
  "https://neuvector-manager:8443/v1/scan/workload/${CONTAINER_ID}/report" \
  -H "X-Auth-Token: ${TOKEN}" | jq '{
    total_vulns: .report.vulnerability | length,
    critical: [.report.vulnerability[] | select(.severity == "Critical")] | length,
    high: [.report.vulnerability[] | select(.severity == "High")] | length,
    medium: [.report.vulnerability[] | select(.severity == "Medium")] | length,
    low: [.report.vulnerability[] | select(.severity == "Low")] | length
  }'
```

A typical vulnerability entry looks like:

```json
{
  "name": "CVE-2023-44487",
  "severity": "High",
  "score": 7.5,
  "vectors": "CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:N/I:N/A:H",
  "package_name": "golang.org/x/net",
  "package_version": "0.8.0",
  "fixed_version": "0.17.0",
  "description": "HTTP/2 Rapid Reset Attack vulnerability",
  "published_timestamp": 1697414400,
  "last_modified_timestamp": 1697500800
}
```

## Step 5: Filter and Export Results

Filter scan results by severity or package:

```bash
# Get only Critical and High vulnerabilities
curl -sk \
  "https://neuvector-manager:8443/v1/scan/workload/${CONTAINER_ID}/report" \
  -H "X-Auth-Token: ${TOKEN}" | \
  jq '[.report.vulnerability[] | select(.severity == "Critical" or .severity == "High")]'

# Export as CSV
curl -sk \
  "https://neuvector-manager:8443/v1/scan/workload/${CONTAINER_ID}/report" \
  -H "X-Auth-Token: ${TOKEN}" | \
  jq -r '.report.vulnerability[] | [.name, .severity, .score, .package_name, .fixed_version] | @csv' \
  > scan-results.csv
```

## Step 6: View Scan Results in the UI

In the NeuVector Manager:

1. Go to **Security Risks** > **Vulnerabilities**
2. Use filters to narrow results by:
   - Severity (Critical, High, Medium, Low)
   - Package name
   - Namespace
   - Workload name
3. Click any CVE to view detailed information including:
   - CVSS score and vector
   - Affected packages
   - Available fixes
   - Impacted containers

## Step 7: Set Up Scan Notifications

Configure alerts when scans find high-severity vulnerabilities:

```bash
# Configure a webhook for scan alerts
curl -sk -X POST \
  "https://neuvector-manager:8443/v1/system/webhook" \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${TOKEN}" \
  -d '{
    "config": {
      "name": "scan-alerts",
      "url": "https://your-webhook.example.com/neuvector",
      "type": "Slack",
      "enable": true,
      "cfg_type": "user"
    }
  }'
```

## Conclusion

NeuVector's container image scanning provides continuous visibility into vulnerabilities across your Kubernetes workloads. By combining auto-scan with admission control and regular registry scans, you create a multi-layered defense that catches vulnerable images both before and during runtime. Use the scan results to prioritize remediation, focusing first on Critical and High severity CVEs with available fixes.
