# How to Set Up Rancher Prime for Enterprise

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Rancher-prime, Enterprise, Setup, Kubernetes

Description: A comprehensive guide to setting up Rancher Prime for enterprise environments, covering licensing, enterprise features, support access, and production hardening.

## Overview

Rancher Prime is SUSE's enterprise-supported distribution of Rancher, providing commercial support, security advisory subscriptions, compliance tooling, and enterprise-grade SLAs. This guide walks through setting up Rancher Prime for production enterprise environments, covering licensing activation, high-availability deployment, and enterprise-specific features.

## What's Included in Rancher Prime

Rancher Prime includes everything in the open-source Rancher community edition plus:

- Enterprise-level support SLAs from SUSE
- Security advisory notifications for critical CVEs
- Priority patch access
- Extended lifecycle support
- FIPS-validated components access
- Rancher Prime audit log with enhanced retention
- NeuVector enterprise integration
- Extended compliance reporting

## Step 1: Prerequisites

Before installing Rancher Prime, prepare your infrastructure:

```bash
# Rancher Prime requires a Kubernetes cluster to run on

# Minimum: 3-node HA cluster (RKE2 recommended)

# Node requirements for production:
# - 4 CPU, 16GB RAM per Rancher management node
# - SSD storage: 50GB+

# Required external dependencies:
# - Load balancer with TLS termination
# - Valid TLS certificate (cert-manager or your PKI)
# - External DNS entry for Rancher hostname
# - S3-compatible object storage for backups
```

## Step 2: Install RKE2 for the Rancher Management Cluster

```yaml
# /etc/rancher/rke2/config.yaml on management cluster nodes
# Use CIS profile for production
profile: cis-1.23
secrets-encryption: true
selinux: true

# HA configuration
server: https://rancher-mgmt-lb.example.com:9345
token: "your-cluster-secret-token"

tls-san:
  - rancher-mgmt-lb.example.com
  - 10.0.1.100   # Load balancer IP
```

```bash
# Install RKE2 on each management node
curl -sfL https://get.rke2.io | INSTALL_RKE2_VERSION=v1.28.6+rke2r1 sh -
systemctl enable --now rke2-server
```

## Step 3: Install cert-manager

```bash
# Install cert-manager for TLS certificate management
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml

# Wait for cert-manager
kubectl -n cert-manager rollout status deploy/cert-manager
kubectl -n cert-manager rollout status deploy/cert-manager-webhook
```

## Step 4: Configure Private Registry Access

Rancher Prime images are served from the SUSE registry. Configure access:

```bash
# Create registry secret for SUSE registry
kubectl create secret docker-registry suse-registry-secret \
  --namespace cattle-system \
  --docker-server=registry.rancher.com \
  --docker-username=your-email@company.com \
  --docker-password=your-suse-portal-password
```

## Step 5: Install Rancher Prime

```bash
# Add Rancher Prime Helm repository
helm repo add rancher-prime https://charts.rancher.com/server-charts/prime
helm repo update

# Create namespace
kubectl create namespace cattle-system

# Install Rancher Prime
helm install rancher rancher-prime/rancher \
  --namespace cattle-system \
  --set hostname=rancher.enterprise.example.com \
  --set replicas=3 \
  --set ingress.tls.source=secret \
  --set privateCA=false \
  --set bootstrapPassword="BootstrapAdminPass123!" \
  --set global.cattle.psp.enabled=false \
  --set rancherImageTag=v2.8.3 \
  --wait

# Verify Rancher is running
kubectl -n cattle-system rollout status deploy/rancher
```

## Step 6: Configure Enterprise Authentication

```text
Rancher UI → Global Settings → Authentication → Active Directory

Settings:
- Server: ldaps://ad.enterprise.com:636
- Domain: enterprise.com
- Service Account: rancher-svc@enterprise.com
- Service Account Password: [from secrets manager]
- User Search Base: OU=Users,DC=enterprise,DC=com
- Group Search Base: OU=Groups,DC=enterprise,DC=com
- Test Connection → Save
```

## Step 7: Configure High Availability for Rancher

```yaml
# Verify Rancher HA deployment
# 3 replicas with pod anti-affinity
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rancher
  namespace: cattle-system
spec:
  replicas: 3
  template:
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app
                    operator: In
                    values: ["rancher"]
              topologyKey: kubernetes.io/hostname
```

## Step 8: Configure Rancher Prime Audit Logging

```bash
# Enable enhanced audit logging in Rancher Prime
kubectl -n cattle-system patch deployment rancher \
  --type=json \
  -p='[
    {
      "op": "add",
      "path": "/spec/template/spec/containers/0/env/-",
      "value": {
        "name": "AUDIT_LEVEL",
        "value": "2"
      }
    },
    {
      "op": "add",
      "path": "/spec/template/spec/containers/0/env/-",
      "value": {
        "name": "AUDIT_LOG_MAXAGE",
        "value": "90"
      }
    }
  ]'
```

## Step 9: Set Up Rancher Backup Operator

```bash
# Install backup operator for enterprise DR
helm install rancher-backup rancher-prime/rancher-backup-crd \
  --namespace cattle-resources-system \
  --create-namespace

helm install rancher-backup rancher-prime/rancher-backup \
  --namespace cattle-resources-system \
  --set persistence.enabled=true \
  --set storageClass=longhorn
```

```yaml
# Configure daily backup to S3
apiVersion: resources.cattle.io/v1
kind: Backup
metadata:
  name: enterprise-daily-backup
  namespace: cattle-resources-system
spec:
  schedule: "0 1 * * *"
  retentionCount: 30   # 30 days retention
  storageLocation:
    s3:
      bucketName: "rancher-prime-backups"
      region: "us-east-1"
      credentialSecretName: s3-backup-credentials
      credentialSecretNamespace: cattle-resources-system
```

## Step 10: Activate Enterprise Support

Register your Rancher Prime installation with SUSE:

```text
1. Log in to https://scc.suse.com with your SUSE account
2. Navigate to Organizations → Your Org → Subscriptions
3. Find your Rancher Prime subscription
4. Copy the registration code
5. In Rancher UI: About → Register → Enter registration code
```

This activates:
- Security advisory notifications
- Access to priority patches
- SLA-backed support tickets

## Step 11: Configure SUSE Security Advisories

```bash
# Configure email notifications for security advisories
# In Rancher UI: Global Settings → Notifications

# Or via Rancher API:
curl -s -k \
  -X PUT \
  -H "Authorization: Bearer ${RANCHER_TOKEN}" \
  -H "Content-Type: application/json" \
  "${RANCHER_URL}/v3/settings/notification-email" \
  -d '{"value": "security-team@enterprise.com"}'
```

## Post-Installation Checklist

- [ ] Rancher Prime installed with 3+ replicas
- [ ] TLS certificate from enterprise CA configured
- [ ] Active Directory / SAML authentication enabled
- [ ] MFA enforced via identity provider
- [ ] Backup operator installed with daily S3 backups
- [ ] Audit logging enabled (level 2+)
- [ ] Enterprise subscription registered with SUSE
- [ ] Security advisory notifications configured
- [ ] Rancher monitoring installed (Prometheus/Grafana)
- [ ] NeuVector installed for runtime security

## Conclusion

Rancher Prime provides enterprise organizations with the commercial support, security advisories, and compliance features needed for production Kubernetes management. The setup process is straightforward but requires careful planning around HA deployment, authentication integration, backup configuration, and subscription activation. Once running, Rancher Prime provides a stable, supported platform for managing your organization's entire Kubernetes fleet.
