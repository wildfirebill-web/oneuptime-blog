# How to Configure Rancher Active-Passive DR

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Disaster-recovery, Active-Passive, Kubernetes, High-Availability

Description: A detailed guide to implementing active-passive disaster recovery for Rancher with automated failover capabilities.

## Introduction

Active-passive DR keeps a warm standby Rancher instance ready to take over when the primary fails. Unlike active-active configurations, this approach is simpler and more cost-effective while still providing strong recovery capabilities.

## Architecture

In an active-passive setup:
- **Active node**: Handles all traffic and operations
- **Passive node**: Stays synchronized via backups, ready to activate on failover
- **Shared storage or replication**: Keeps both nodes current

## Prerequisites

- Primary Rancher cluster (production)
- Secondary server/cluster for passive instance
- S3 bucket for backup storage
- Network connectivity between sites
- DNS with low TTL (60 seconds)

## Step 1: Configure Automated Backups on Primary

```yaml
# primary-backup-config.yaml

apiVersion: resources.cattle.io/v1
kind: Backup
metadata:
  name: active-passive-backup
  namespace: cattle-resources-system
spec:
  storageLocation:
    s3:
      bucketName: rancher-ap-dr
      folder: primary
      region: us-east-1
      endpoint: s3.amazonaws.com
      credentialSecretName: s3-credentials
      credentialSecretNamespace: cattle-resources-system
  schedule: "*/30 * * * *"  # Every 30 minutes for low RPO
  retentionCount: 96         # 48 hours of 30-min backups
  encryptionConfigSecretName: encryption-config
```

Apply encryption secret:

```bash
# Create encryption key
kubectl create secret generic encryption-config \
  --namespace cattle-resources-system \
  --from-literal=encryptionConfig='{"encryptionKey":"your-32-char-encryption-key-here"}'
```

## Step 2: Prepare Passive Node

```bash
#!/bin/bash
# setup-passive.sh - Run on passive server

# Install RKE2
curl -sfL https://get.rke2.io | sh -
systemctl enable rke2-server.service
systemctl start rke2-server.service

# Wait for RKE2 to be ready
until kubectl --kubeconfig /etc/rancher/rke2/rke2.yaml get nodes; do
  echo "Waiting for RKE2..."
  sleep 10
done

# Install cert-manager
helm repo add jetstack https://charts.jetstack.io && helm repo update
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace \
  --set installCRDs=true

# Install Rancher Backup Operator (needed for restore)
helm repo add rancher-charts https://charts.rancher.io && helm repo update
helm install rancher-backup rancher-charts/rancher-backup \
  --namespace cattle-resources-system --create-namespace
```

## Step 3: Configure Passive Node Monitoring

Create a script that monitors the active node and triggers failover:

```bash
#!/bin/bash
# monitor-active.sh - Run on passive node as cron job

ACTIVE_URL="https://rancher.example.com"
PASSIVE_RANCHER_URL="https://rancher-passive.example.com"
CHECK_INTERVAL=30
FAILURE_THRESHOLD=3
failure_count=0

while true; do
  if curl -sf --max-time 10 "${ACTIVE_URL}/v3/ping" > /dev/null 2>&1; then
    echo "$(date): Active node is healthy"
    failure_count=0
  else
    failure_count=$((failure_count + 1))
    echo "$(date): Active node check failed ($failure_count/$FAILURE_THRESHOLD)"
    
    if [ $failure_count -ge $FAILURE_THRESHOLD ]; then
      echo "$(date): FAILURE THRESHOLD REACHED - Initiating failover"
      /usr/local/bin/failover.sh
      break
    fi
  fi
  sleep $CHECK_INTERVAL
done
```

## Step 4: Create Failover Script

```bash
#!/bin/bash
# failover.sh - Execute to promote passive to active

set -e
echo "=== RANCHER FAILOVER INITIATED ==="
echo "Time: $(date)"

# 1. Get latest backup
echo "Fetching latest backup..."
LATEST_BACKUP=$(aws s3 ls s3://rancher-ap-dr/primary/ \
  --recursive | sort | tail -1 | awk '{print $4}')

if [ -z "$LATEST_BACKUP" ]; then
  echo "ERROR: No backup found!"
  exit 1
fi

echo "Latest backup: $LATEST_BACKUP"

# 2. Restore Rancher from backup
kubectl apply -f - << RESTOREEOF
apiVersion: resources.cattle.io/v1
kind: Restore
metadata:
  name: failover-$(date +%Y%m%d%H%M%S)
  namespace: cattle-resources-system
spec:
  backupFilename: ${LATEST_BACKUP}
  prune: false
  storageLocation:
    s3:
      bucketName: rancher-ap-dr
      folder: primary
      region: us-east-1
      credentialSecretName: s3-credentials
RESTOREEOF

# 3. Wait for restore to complete
echo "Waiting for restore to complete..."
kubectl wait restore --all \
  --for=condition=Ready \
  --timeout=600s \
  --namespace cattle-resources-system

# 4. Notify team
curl -X POST "$SLACK_WEBHOOK" \
  -H 'Content-type: application/json' \
  --data '{"text":"RANCHER FAILOVER COMPLETE: Passive node is now active!"}'

echo "=== FAILOVER COMPLETE ==="
```

## Step 5: Configure DNS Failover

```bash
# Update DNS when failover occurs
# For AWS Route53:
aws route53 change-resource-record-sets \
  --hosted-zone-id YOUR_ZONE_ID \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "rancher.example.com",
        "Type": "A",
        "TTL": 60,
        "ResourceRecords": [
          {"Value": "PASSIVE_NODE_IP"}
        ]
      }
    }]
  }'
```

## Step 6: Validate Passive Node Readiness

Run these checks weekly to ensure the passive node stays ready:

```bash
#!/bin/bash
# validate-passive.sh

echo "=== Passive Node Readiness Check ==="

# Check S3 backup accessibility
echo "Checking S3 backup access..."
aws s3 ls s3://rancher-ap-dr/primary/ | tail -5

# Check latest backup age
LATEST_BACKUP_TIME=$(aws s3 ls s3://rancher-ap-dr/primary/ \
  --recursive | sort | tail -1 | awk '{print $1" "$2}')
echo "Latest backup: $LATEST_BACKUP_TIME"

# Check passive Rancher health
echo "Checking passive Rancher health..."
kubectl --context passive-cluster get pods -n cattle-system

# Check cert-manager
kubectl --context passive-cluster get pods -n cert-manager

echo "=== Readiness Check Complete ==="
```

## Conclusion

Active-passive DR provides a cost-effective way to maintain business continuity for Rancher. The key is frequent backups to shared storage, a reliable monitoring mechanism on the passive side, and a well-tested failover script. Regular drills ensure the passive node can actually take over within your RTO when needed.
