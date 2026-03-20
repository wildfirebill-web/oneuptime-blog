# How to Configure NeuVector Registry Scanning

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: NeuVector, Registry Scanning, Container Security, Vulnerability Management, Kubernetes

Description: Configure NeuVector to automatically scan container registries for vulnerabilities across all stored images on a scheduled basis.

## Introduction

Registry scanning in NeuVector allows you to periodically scan all images stored in your container registries — not just the ones currently running. This gives you a complete inventory of vulnerabilities across your entire image library, enabling proactive remediation before images are deployed.

## Prerequisites

- NeuVector with Scanner component running
- Access credentials for your container registries
- NeuVector Manager access

## Step 1: Configure Registry Credentials

### Add Docker Hub Registry

In the NeuVector UI:
1. Go to **Assets** > **Registries**
2. Click **Add Registry**
3. Fill in the form:

```
Name: dockerhub
Registry: https://registry-1.docker.io
Username: your-dockerhub-username
Password: your-dockerhub-token
Scan Layers: Enabled
Rescan After CVE DB Update: Enabled
```

Via API:

```bash
# Add Docker Hub registry
curl -sk -X POST \
  "https://neuvector-manager:8443/v1/scan/registry" \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${TOKEN}" \
  -d '{
    "config": {
      "name": "dockerhub",
      "registry": "https://registry-1.docker.io",
      "auth_token": "",
      "username": "your-username",
      "password": "your-token",
      "scan_layers": true,
      "rescan_after_db_update": true,
      "cfg_type": "user"
    }
  }'
```

### Add Private Registry (e.g., Harbor)

```bash
# Add private Harbor registry
curl -sk -X POST \
  "https://neuvector-manager:8443/v1/scan/registry" \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${TOKEN}" \
  -d '{
    "config": {
      "name": "harbor-prod",
      "registry": "https://harbor.company.com",
      "username": "scan-robot",
      "password": "robot-token",
      "filters": ["production/*", "staging/*"],
      "scan_layers": true,
      "rescan_after_db_update": true,
      "schedule": {
        "schedule": "daily",
        "interval": 0
      },
      "cfg_type": "user"
    }
  }'
```

### Add Amazon ECR Registry

```bash
# Add AWS ECR registry (uses IAM role or access keys)
curl -sk -X POST \
  "https://neuvector-manager:8443/v1/scan/registry" \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${TOKEN}" \
  -d '{
    "config": {
      "name": "aws-ecr",
      "registry": "https://123456789.dkr.ecr.us-east-1.amazonaws.com",
      "auth_with_key": true,
      "access_key_id": "AKIAIOSFODNN7EXAMPLE",
      "secret_access_key": "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY",
      "aws_region": "us-east-1",
      "scan_layers": true,
      "rescan_after_db_update": true,
      "schedule": {
        "schedule": "daily",
        "interval": 0
      }
    }
  }'
```

### Add Google Artifact Registry

```bash
# Add Google Artifact Registry
curl -sk -X POST \
  "https://neuvector-manager:8443/v1/scan/registry" \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${TOKEN}" \
  -d '{
    "config": {
      "name": "google-gar",
      "registry": "https://us-central1-docker.pkg.dev",
      "auth_token": "$(gcloud auth print-access-token)",
      "filters": ["my-project/production/*"],
      "scan_layers": true,
      "rescan_after_db_update": true
    }
  }'
```

## Step 2: Configure Scan Filters

Use filters to control which repositories and tags are scanned:

```bash
# Update registry with specific filters
curl -sk -X PATCH \
  "https://neuvector-manager:8443/v1/scan/registry/harbor-prod" \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${TOKEN}" \
  -d '{
    "config": {
      "filters": [
        "production/myapp:*",
        "production/nginx:*",
        "staging/myapp:latest"
      ]
    }
  }'
```

## Step 3: Schedule Automatic Scans

Configure scan schedules:

```bash
# Schedule daily scans at midnight
curl -sk -X PATCH \
  "https://neuvector-manager:8443/v1/scan/registry/harbor-prod" \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${TOKEN}" \
  -d '{
    "config": {
      "schedule": {
        "schedule": "daily",
        "interval": 0
      }
    }
  }'

# Schedule weekly scans
curl -sk -X PATCH \
  "https://neuvector-manager:8443/v1/scan/registry/harbor-prod" \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${TOKEN}" \
  -d '{
    "config": {
      "schedule": {
        "schedule": "weekly",
        "interval": 0
      }
    }
  }'
```

## Step 4: Trigger a Manual Scan

```bash
# Start a manual scan of a registry
curl -sk -X POST \
  "https://neuvector-manager:8443/v1/scan/registry/harbor-prod/scan" \
  -H "X-Auth-Token: ${TOKEN}"

# Check scan status
curl -sk \
  "https://neuvector-manager:8443/v1/scan/registry/harbor-prod" \
  -H "X-Auth-Token: ${TOKEN}" | jq '{
    name: .config.name,
    status: .status.status,
    scanned: .status.scanned,
    failed: .status.failed,
    total: .status.total
  }'
```

## Step 5: View Registry Scan Results

```bash
# Get scan results for all images in a registry
curl -sk \
  "https://neuvector-manager:8443/v1/scan/registry/harbor-prod/images" \
  -H "X-Auth-Token: ${TOKEN}" | jq '.images[] | {
    image_id: .image_id,
    repository: .repository,
    tag: .tag,
    critical: .critical,
    high: .high,
    medium: .medium,
    low: .low,
    scan_date: .scanned_at
  }'
```

In the UI:
1. Go to **Assets** > **Registries**
2. Click a registry name to see all scanned images
3. Click an image to view detailed vulnerability report

## Step 6: Export Registry Scan Report

```bash
# Export all high and critical CVEs from a registry
curl -sk \
  "https://neuvector-manager:8443/v1/scan/registry/harbor-prod/images" \
  -H "X-Auth-Token: ${TOKEN}" | \
  jq -r '.images[] |
    .repository + ":" + .tag + "," +
    (.critical|tostring) + "," +
    (.high|tostring) + "," +
    (.medium|tostring)' | \
  awk 'BEGIN {print "Image,Critical,High,Medium"} {print}' > registry-scan-summary.csv
```

## Step 7: Configure Rescan on CVE Database Update

Enable automatic rescanning when the CVE database is updated:

```bash
curl -sk -X PATCH \
  "https://neuvector-manager:8443/v1/scan/registry/harbor-prod" \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${TOKEN}" \
  -d '{
    "config": {
      "rescan_after_db_update": true
    }
  }'
```

## Conclusion

Registry scanning gives you a complete picture of vulnerabilities across all your stored images, not just those currently running. By scheduling regular scans with automatic rescan on CVE database updates, you ensure that images are continuously evaluated against the latest threat intelligence. Use registry scan results to prioritize image updates, remove outdated images, and prevent vulnerable images from being used in new deployments.
