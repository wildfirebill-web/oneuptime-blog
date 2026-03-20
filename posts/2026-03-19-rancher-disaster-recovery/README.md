# How to Set Up Disaster Recovery for Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Disaster Recovery, Backup, Restore

Description: Learn how to design and implement a comprehensive disaster recovery plan for your Rancher management server and managed clusters.

A well-designed disaster recovery (DR) plan for Rancher ensures you can recover from catastrophic failures with minimal downtime and data loss. This guide covers building a complete DR strategy including backup automation, recovery procedures, and regular testing.

## Prerequisites

- Rancher v2.5 or later in production
- The Rancher Backup Operator installed
- External storage (S3 or equivalent) in a different region or data center
- A secondary environment for DR testing
- kubectl and Helm 3 access

## Step 1: Define Recovery Objectives

Before configuring DR, establish your objectives:

- **Recovery Point Objective (RPO)**: Maximum acceptable data loss measured in time. For example, an RPO of 1 hour means you need backups at least every hour.
- **Recovery Time Objective (RTO)**: Maximum acceptable downtime. For example, an RTO of 30 minutes means you must be able to restore within half an hour.

These objectives drive your backup frequency and recovery procedures.

## Step 2: Configure Multi-Tier Backup Schedule

Create a layered backup strategy that balances storage costs with recovery granularity. Save the following as `dr-backups.yaml`:

```yaml
apiVersion: resources.cattle.io/v1
kind: Backup
metadata:
  name: dr-hourly-backup
spec:
  resourceSetName: rancher-resource-set
  retentionCount: 24
  schedule: "0 * * * *"
  encryptionConfigSecretName: backup-encryption
  storageLocation:
    s3:
      bucketName: rancher-dr-backups
      folder: hourly
      endpoint: s3.amazonaws.com
      region: us-west-2
      credentialSecretName: dr-s3-creds
      credentialSecretNamespace: cattle-resources-system
---
apiVersion: resources.cattle.io/v1
kind: Backup
metadata:
  name: dr-daily-backup
spec:
  resourceSetName: rancher-resource-set
  retentionCount: 30
  schedule: "0 1 * * *"
  encryptionConfigSecretName: backup-encryption
  storageLocation:
    s3:
      bucketName: rancher-dr-backups
      folder: daily
      endpoint: s3.amazonaws.com
      region: us-west-2
      credentialSecretName: dr-s3-creds
      credentialSecretNamespace: cattle-resources-system
---
apiVersion: resources.cattle.io/v1
kind: Backup
metadata:
  name: dr-weekly-backup
spec:
  resourceSetName: rancher-resource-set
  retentionCount: 12
  schedule: "0 2 * * 0"
  encryptionConfigSecretName: backup-encryption
  storageLocation:
    s3:
      bucketName: rancher-dr-backups
      folder: weekly
      endpoint: s3.amazonaws.com
      region: us-west-2
      credentialSecretName: dr-s3-creds
      credentialSecretNamespace: cattle-resources-system
```

Apply all backup schedules:

```bash
kubectl apply -f dr-backups.yaml
```

## Step 3: Set Up Cross-Region Backup Replication

Enable S3 cross-region replication to ensure backups survive a regional outage:

```bash
aws s3api put-bucket-replication \
  --bucket rancher-dr-backups \
  --replication-configuration '{
    "Role": "arn:aws:iam::ACCOUNT_ID:role/s3-replication-role",
    "Rules": [
      {
        "Status": "Enabled",
        "Priority": 1,
        "Filter": {"Prefix": ""},
        "Destination": {
          "Bucket": "arn:aws:s3:::rancher-dr-backups-replica",
          "StorageClass": "STANDARD_IA"
        },
        "DeleteMarkerReplication": {"Status": "Enabled"}
      }
    ]
  }'
```

## Step 4: Document the Recovery Procedure

Create a runbook that your team can follow during a disaster. The key steps are:

1. Provision a new Kubernetes cluster in the DR region.
2. Install cert-manager and Rancher (same version).
3. Install the Backup Operator.
4. Create storage credentials.
5. Scale down Rancher.
6. Restore from the latest backup.
7. Scale up Rancher.
8. Update DNS.
9. Verify downstream clusters reconnect.

Save this as a script for rapid execution. Here is an example `dr-recover.sh`:

```bash
#!/bin/bash
set -e

BACKUP_FILE=$1
RANCHER_VERSION=$2
HOSTNAME=$3

echo "Starting Rancher DR recovery..."

# Install cert-manager

kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.3/cert-manager.yaml
kubectl wait --for=condition=Available -n cert-manager deployment/cert-manager --timeout=120s

# Install Rancher
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
helm repo update
helm install rancher rancher-latest/rancher \
  -n cattle-system --create-namespace \
  --set hostname=$HOSTNAME \
  --set replicas=3 \
  --version=$RANCHER_VERSION

kubectl rollout status deployment rancher -n cattle-system

# Install Backup Operator
helm install rancher-backup-crd rancher-charts/rancher-backup-crd \
  -n cattle-resources-system --create-namespace
helm install rancher-backup rancher-charts/rancher-backup \
  -n cattle-resources-system

# Wait for operator
kubectl rollout status deployment rancher-backup -n cattle-resources-system

# Scale down Rancher for restore
kubectl scale deployment rancher -n cattle-system --replicas=0

echo "Apply the restore resource manually with backup file: $BACKUP_FILE"
echo "Then scale Rancher back up with: kubectl scale deployment rancher -n cattle-system --replicas=3"
```

## Step 5: Set Up Monitoring and Alerts

Monitor your backup status and alert on failures:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: dr-backup-alerts
  namespace: cattle-monitoring-system
spec:
  groups:
  - name: disaster-recovery
    rules:
    - alert: BackupMissed
      expr: |
        time() - max(kube_customresource_status_condition_last_transition_time{
          group="resources.cattle.io",
          kind="Backup",
          condition="Ready",
          status="True"
        }) > 7200
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "No successful Rancher backup in the last 2 hours"
    - alert: BackupFailed
      expr: |
        kube_customresource_status_condition{
          group="resources.cattle.io",
          kind="Backup",
          condition="Ready",
          status="False"
        } == 1
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Rancher backup has failed"
```

## Step 6: Test Recovery Regularly

Schedule regular DR tests, ideally quarterly. A DR test involves:

1. Provisioning a test cluster.
2. Running the recovery procedure against the latest backup.
3. Verifying all Rancher resources are restored correctly.
4. Measuring actual RTO and comparing it to your target.
5. Documenting any issues found and updating the runbook.
6. Tearing down the test cluster.

## Step 7: Protect Downstream Clusters

Rancher backups protect the management plane, but downstream cluster workloads need their own backup strategy:

- Enable etcd snapshots on all managed clusters (covered in the etcd backup guide).
- Use Velero or similar tools for workload-level backups.
- Store downstream backups in the same cross-region storage for consistency.

## Step 8: Secure DR Credentials

Store DR credentials in a separate secure location:

- Use a secrets manager (HashiCorp Vault, AWS Secrets Manager) for encryption keys and storage credentials.
- Ensure the DR team has access to credentials even if the primary infrastructure is unavailable.
- Rotate credentials regularly and update both primary and DR configurations.

## Conclusion

Disaster recovery for Rancher requires a layered approach combining automated backups with documented recovery procedures and regular testing. By establishing clear RPO and RTO targets, configuring multi-tier backup schedules with cross-region replication, and testing your recovery process regularly, you can ensure your Rancher management server can be recovered quickly and reliably from any failure scenario.
