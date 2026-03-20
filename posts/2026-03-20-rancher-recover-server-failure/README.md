# How to Recover Rancher After Complete Server Failure

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Disaster-recovery, Recovery, Server-failure, Kubernetes

Description: Step-by-step recovery guide for restoring Rancher after a complete server failure using backup files and the restore operator.

## Introduction

When your Rancher server suffers a complete failure-hardware crash, OS corruption, or accidental deletion-you need to restore from backup quickly. This guide walks through the complete recovery process from a bare metal or fresh cloud instance.

## What Gets Lost Without Backup

- All cluster registrations and configurations
- Users, roles, and RBAC settings
- Projects, namespaces, and their policies
- Secrets and credentials stored in Rancher
- Catalogs and app configurations
- Alert, monitoring, and logging configurations

Note: Downstream cluster workloads continue running even when Rancher is down-only management plane features are affected.

## Pre-Recovery Checklist

```bash
# Before starting recovery, gather:

echo "Required items:"
echo "1. Backup file location (S3 bucket, NFS path)"
echo "2. Backup encryption key (from secure storage)"
echo "3. Original Rancher hostname"
echo "4. SSL certificates or Let's Encrypt"
echo "5. New server: 4 CPU, 16GB RAM minimum"
echo "   Clean OS: Ubuntu 22.04 or RHEL 8/9"
```

## Step 1: Provision New Server

```bash
# Update system packages
sudo apt-get update && sudo apt-get upgrade -y

# Install required dependencies
sudo apt-get install -y curl wget apt-transport-https

# Verify system requirements
echo "CPU cores: $(nproc)"
echo "RAM: $(free -h | awk '/^Mem:/ {print $2}')"
echo "Disk: $(df -h / | awk 'NR==2 {print $4}') available"
```

## Step 2: Install Kubernetes (RKE2)

```bash
# Install RKE2
curl -sfL https://get.rke2.io | \
  INSTALL_RKE2_VERSION="v1.28.8+rke2r1" sh -

# Configure RKE2
mkdir -p /etc/rancher/rke2
cat > /etc/rancher/rke2/config.yaml << 'CONFIG'
node-name: rancher-server
tls-san:
  - rancher.example.com
  - 10.0.1.100
cni: calico
CONFIG

# Start RKE2
systemctl enable rke2-server.service
systemctl start rke2-server.service

export KUBECONFIG=/etc/rancher/rke2/rke2.yaml
```

## Step 3: Install Prerequisites

```bash
# Install Helm
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Install cert-manager
helm repo add jetstack https://charts.jetstack.io && helm repo update

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.13.0 \
  --set installCRDs=true

kubectl wait pods -n cert-manager \
  --all --for=condition=Ready \
  --timeout=300s
```

## Step 4: Install Rancher Backup Operator First

Install the backup operator before Rancher to enable restore:

```bash
helm repo add rancher-charts https://charts.rancher.io && helm repo update

helm install rancher-backup rancher-charts/rancher-backup \
  --namespace cattle-resources-system \
  --create-namespace \
  --set image.tag=v4.0.0

kubectl wait pods -n cattle-resources-system \
  --all --for=condition=Ready \
  --timeout=300s
```

## Step 5: Restore S3 Credentials

```bash
# Recreate S3 credentials secret
kubectl create secret generic rancher-backup-s3-creds \
  --namespace cattle-resources-system \
  --from-literal=accessKey="YOUR_ACCESS_KEY" \
  --from-literal=secretKey="YOUR_SECRET_KEY"

# Recreate encryption config
kubectl create secret generic backup-encryption-key \
  --namespace cattle-resources-system \
  --from-literal=encryptionConfig='{"encryptionKey":"YOUR_ENCRYPTION_KEY"}'
```

## Step 6: Execute the Restore

```yaml
# restore.yaml
apiVersion: resources.cattle.io/v1
kind: Restore
metadata:
  name: rancher-recovery
  namespace: cattle-resources-system
spec:
  backupFilename: rancher/rancher-backup-2026-03-19T02-00-00Z.tar.gz
  prune: true
  storageLocation:
    s3:
      bucketName: rancher-production-backups
      folder: rancher
      region: us-east-1
      endpoint: s3.amazonaws.com
      credentialSecretName: rancher-backup-s3-creds
      credentialSecretNamespace: cattle-resources-system
  encryptionConfigSecretName: backup-encryption-key
```

```bash
# Apply restore and monitor progress
kubectl apply -f restore.yaml
kubectl get restore -n cattle-resources-system -w
```

## Step 7: Install Rancher After Restore

```bash
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
helm repo update

helm install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --create-namespace \
  --set hostname=rancher.example.com \
  --set bootstrapPassword=your-bootstrap-password \
  --set replicas=1 \
  --set ingress.tls.source=letsEncrypt \
  --set letsEncrypt.email=admin@example.com

kubectl wait deployment/rancher \
  --namespace cattle-system \
  --for=condition=Available \
  --timeout=600s
```

## Step 8: Verify Recovery

```bash
#!/bin/bash
RANCHER_URL="https://rancher.example.com"

# Check API
curl -sf "${RANCHER_URL}/v3/ping" && echo "API: OK" || echo "API: FAILED"

# Verify clusters are visible
kubectl get clusters.management.cattle.io

# Check Fleet status
kubectl get pods -n cattle-fleet-system

# Check downstream cluster connectivity
kubectl get clusters.management.cattle.io \
  -o custom-columns='NAME:.metadata.name,READY:.status.conditions[0].status'
```

## Post-Recovery Steps

1. Verify DNS points to new server IP
2. Test user logins and LDAP/OIDC authentication
3. Check downstream cluster agent connectivity
4. Verify monitoring and logging configurations
5. Document the incident and recovery timeline

## Conclusion

Recovering from a complete Rancher server failure is straightforward when you have reliable backups and a tested recovery procedure. The key is having the backup operator installed before Rancher and keeping your encryption keys safely stored outside the Rancher environment. With regular backups and this recovery procedure, you can restore a complete Rancher environment in under an hour.
