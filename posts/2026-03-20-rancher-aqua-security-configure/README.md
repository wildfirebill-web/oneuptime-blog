# How to Configure Rancher with Aqua Security

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Aqua-security, Container-security, Kubernetes, Vulnerability-Scanning

Description: A step-by-step guide to deploying and configuring Aqua Security on Rancher-managed Kubernetes clusters for container image scanning, runtime protection, and compliance.

## Overview

Aqua Security is a comprehensive Cloud-Native Application Protection Platform (CNAPP) that provides container image scanning, Kubernetes runtime security, network policies, and compliance reporting. This guide covers deploying the Aqua Platform on Rancher-managed clusters, configuring image scanning, and setting up runtime enforcement.

## Prerequisites

- Aqua Security license and account at https://portal.aquasec.com
- Rancher v2.7+ with RKE2 or K3s clusters
- PostgreSQL database for Aqua server (or use embedded)
- TLS certificate for Aqua web UI

## Step 1: Deploy Aqua Server

```bash
# Add Aqua Helm repository

helm repo add aqua https://helm.aquasec.com
helm repo update

# Create namespace and registry secret
kubectl create namespace aqua
kubectl create secret docker-registry aqua-registry \
  --namespace aqua \
  --docker-server=registry.aquasec.com \
  --docker-username="${AQUA_REGISTRY_USERNAME}" \
  --docker-password="${AQUA_REGISTRY_PASSWORD}"
```

```yaml
# aqua-server-values.yaml
imageCredentials:
  create: false
  name: aqua-registry

db:
  external:
    enabled: true
    name: aqua
    host: postgres.aqua.svc
    port: 5432
    user: aqua
    password: "${AQUA_DB_PASSWORD}"

server:
  image:
    repository: registry.aquasec.com/console
    tag: "2024.4"
  service:
    type: LoadBalancer
  # Admin credentials
  admin:
    token: "${AQUA_ADMIN_TOKEN}"
    password: "${AQUA_ADMIN_PASSWORD}"

license:
  token: "${AQUA_LICENSE_TOKEN}"
```

```bash
helm install aqua aqua/aqua \
  --namespace aqua \
  --values aqua-server-values.yaml
```

## Step 2: Deploy Aqua Enforcers on Each Cluster

The Aqua Enforcer runs as a DaemonSet and provides runtime security:

```bash
# Get enforcer token from Aqua UI: Administration → Enforcers → Add Enforcer

helm install aqua-enforcer aqua/enforcer \
  --namespace aqua \
  --set token="${AQUA_ENFORCER_TOKEN}" \
  --set gateway.host=aqua-gateway.aqua.svc \
  --set gateway.port=8443 \
  --set enforcerMode=enforce    # or "audit" for monitoring only
```

```yaml
# Enforcer DaemonSet verification
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: aqua-agent
  namespace: aqua
spec:
  selector:
    matchLabels:
      app: aqua-agent
  template:
    spec:
      hostPID: true
      containers:
        - name: aqua-agent
          image: registry.aquasec.com/enforcer:2024.4
          securityContext:
            privileged: false
            capabilities:
              add:
                - SYS_ADMIN
                - NET_ADMIN
                - NET_RAW
```

## Step 3: Configure Image Scanning

### Registry Integration

Connect Aqua to your container registries:

```bash
# Add registry via Aqua CLI
aquactl registry add \
  --name harbor-internal \
  --type Harbor \
  --url https://harbor.example.com \
  --username aqua-scanner \
  --password "${HARBOR_PASSWORD}"

# Trigger a registry scan
aquactl scan registry scan --name harbor-internal
```

### CI/CD Pipeline Integration

```yaml
# GitHub Actions: Scan image with Aqua before push
name: Build and Security Scan
on: [push]

jobs:
  build-and-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build image
        run: docker build -t myapp:${{ github.sha }} .

      - name: Aqua Security Scan
        env:
          AQUA_SERVER: ${{ secrets.AQUA_SERVER }}
          AQUA_TOKEN: ${{ secrets.AQUA_TOKEN }}
        run: |
          # Download and run trivy-cli (Aqua's scanner)
          docker run --rm \
            -v /var/run/docker.sock:/var/run/docker.sock \
            -e AQUA_SERVER="${AQUA_SERVER}" \
            -e AQUA_TOKEN="${AQUA_TOKEN}" \
            registry.aquasec.com/scanner:2024.4 \
            --host ${AQUA_SERVER} \
            --user scanner \
            --password ${AQUA_TOKEN} \
            --register-compliant \
            --jsonfile scan-results.json \
            docker:myapp:${{ github.sha }}

      - name: Check scan results
        run: |
          CRITICAL=$(jq '.vulnerability_summary.critical' scan-results.json)
          if [ "${CRITICAL}" -gt 0 ]; then
            echo "FAIL: ${CRITICAL} critical vulnerabilities found"
            exit 1
          fi
```

## Step 4: Configure Runtime Policies

```yaml
# Aqua container policy via API
# Block containers from running as root
POST /api/v1/containerpolicies
{
  "name": "production-policy",
  "description": "Production runtime policy",
  "runtime_mode": "enforce",
  "block_privileged_containers": true,
  "block_root_user": true,
  "block_non_compliant_images": true,
  "audit_success": false,
  "containers_allowed": ["registry.example.com/*"],
  "block_exec_console": false,
  "drift_prevention": {
    "enabled": true,
    "exec_allowed": false,
    "bypass_scope": {
      "enabled": false
    }
  },
  "network": {
    "block_metadata_service": true,
    "allow_internal_networking": true
  }
}
```

## Step 5: Kubernetes Assurance Policies

```yaml
# Aqua Kubernetes Assurance Policy - block non-compliant workloads
# Configure via Aqua UI: Policies → Kubernetes Assurance

# Example checks:
# - Pod Security Admission level: restricted
# - Container runs as non-root
# - No host namespace sharing
# - Resource limits required
# - Image from approved registry only
# - No latest tag
```

## Step 6: Configure NVD/CVE Feeds

```bash
# Aqua continuously updates its CVE database
# Verify feed update status:
curl -H "Authorization: Bearer ${AQUA_TOKEN}" \
  https://aqua.example.com/api/v1/vulnerability/nvd_feed \
  | jq '{status: .status, last_update: .last_update}'
```

## Step 7: Compliance Reporting

```bash
# Generate CIS Docker/Kubernetes benchmark report
curl -H "Authorization: Bearer ${AQUA_TOKEN}" \
  -X POST https://aqua.example.com/api/v1/compliance/check \
  -d '{
    "compliance_type": "kubernetes_cis",
    "cluster_id": "prod-cluster-01"
  }'

# Download report
curl -H "Authorization: Bearer ${AQUA_TOKEN}" \
  "https://aqua.example.com/api/v1/reports/compliance/prod-cluster-01" \
  -o compliance-report.pdf
```

## Step 8: Alert Integration

```yaml
# Configure Aqua to send alerts to your monitoring system
# Aqua UI: Administration → Integrations → Webhooks

POST /api/v1/settings/integrations
{
  "type": "webhook",
  "name": "slack-alerts",
  "url": "https://hooks.slack.com/services/xxx",
  "events": [
    "scan_failed",
    "runtime_incident",
    "non_compliant_workload",
    "drift_detected"
  ]
}
```

## Conclusion

Integrating Aqua Security with Rancher provides enterprise-grade container security across the full application lifecycle. Image scanning in CI/CD pipelines catches vulnerabilities before deployment, Aqua Enforcers on Rancher-managed clusters provide runtime protection, and compliance reporting keeps your security posture visible. For organizations using Rancher without the SUSE stack (NeuVector), Aqua Security is a strong commercial alternative that covers both container and cloud security requirements.
