# How to Implement Backup Best Practices in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Backup, Velero, Disaster Recovery, Kubernetes, etcd

Description: Implement backup best practices in Rancher covering etcd backups, Rancher management plane backups, application data with Velero, and disaster recovery testing to ensure business continuity.

## Introduction

Backup strategy in Rancher covers three distinct layers: the Rancher management plane (including cluster configurations), Kubernetes etcd (cluster state), and application data (PVCs, databases). Each layer requires different backup tools and frequencies. Without all three layers covered, disaster recovery will be incomplete.

## Backup Layers

| Layer | What to back up | Tool | Frequency |
|---|---|---|---|
| Rancher management plane | Cluster configs, RBAC, Projects | Rancher Backup Operator | Daily |
| Kubernetes etcd | Cluster state, deployments | RKE2 built-in / Velero | Every 6 hours |
| Application data | PVCs, databases | Velero + DB-specific tools | Daily / hourly |

## Step 1: Rancher Management Plane Backup

```bash
# Install Rancher Backup Operator

helm install rancher-backup rancher-charts/rancher-backup \
  --namespace cattle-resources-system \
  --create-namespace \
  --set persistence.enabled=true \
  --set persistence.storageClass=longhorn

# Create S3-backed backup location
kubectl apply -f - <<EOF
apiVersion: resources.cattle.io/v1
kind: BackupStorageLocation
metadata:
  name: s3-backup
  namespace: cattle-resources-system
spec:
  storageProvider: s3
  bucketName: rancher-management-backups
  region: us-east-1
  accessKey:
    secretName: aws-credentials
    secretKey: access-key
  secretKey:
    secretName: aws-credentials
    secretKey: secret-key
EOF

# Create scheduled backup
kubectl apply -f - <<EOF
apiVersion: resources.cattle.io/v1
kind: Backup
metadata:
  name: rancher-daily
  namespace: cattle-resources-system
spec:
  storageLocation:
    name: s3-backup
  schedule: "0 2 * * *"      # Daily at 2 AM UTC
  retentionCount: 14           # Keep 14 days
  encryptionConfigSecretName: backup-encryption-secret
EOF
```

## Step 2: etcd Backup

RKE2 automatically backs up etcd:

```yaml
# /etc/rancher/rke2/config.yaml
etcd-snapshot-schedule-cron: "0 */6 * * *"    # Every 6 hours
etcd-snapshot-retention: 10                    # Keep 10 snapshots
etcd-snapshot-dir: /var/lib/rancher/rke2/server/db/snapshots
# Optionally upload to S3
etcd-s3: true
etcd-s3-bucket: my-etcd-backups
etcd-s3-region: us-east-1
etcd-s3-access-key: <access-key>
etcd-s3-secret-key: <secret-key>
```

```bash
# Manually trigger an etcd snapshot
rke2 etcd-snapshot save --name pre-upgrade-backup

# List available snapshots
rke2 etcd-snapshot list

# Restore from snapshot (on single-node, for HA use Rancher restore)
rke2 server --cluster-reset --cluster-reset-restore-path=/path/to/snapshot
```

## Step 3: Application Data Backup with Velero

```bash
# Install Velero
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.7.0 \
  --bucket my-velero-backups \
  --secret-file ./credentials-velero \
  --backup-location-config region=us-east-1 \
  --snapshot-location-config region=us-east-1 \
  --use-volume-snapshots=true

# Create daily backup schedule
velero schedule create daily-production \
  --schedule="0 3 * * *" \
  --include-namespaces production \
  --ttl 720h0m0s             # Retain for 30 days

# Backup before upgrades
velero backup create pre-upgrade-$(date +%Y%m%d) \
  --include-namespaces production,staging \
  --wait
```

## Step 4: Database-Specific Backups

For databases, use application-consistent backups in addition to Velero:

```yaml
# CronJob for PostgreSQL backup
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-backup
  namespace: production
spec:
  schedule: "0 1 * * *"    # Daily at 1 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: pg-backup
              image: postgres:15
              command:
                - /bin/sh
                - -c
                - |
                  pg_dump -h postgres-svc -U $PGUSER $PGDATABASE | \
                  gzip | \
                  aws s3 cp - s3://db-backups/postgres-$(date +%Y%m%d).sql.gz
              env:
                - name: PGPASSWORD
                  valueFrom:
                    secretKeyRef:
                      name: postgres-secret
                      key: password
          restartPolicy: OnFailure
```

## Step 5: Test Your Backups

```bash
# Monthly DR drill: restore Rancher in a test environment
# 1. Create a test Rancher environment
# 2. Restore from backup
kubectl apply -f - <<EOF
apiVersion: resources.cattle.io/v1
kind: Restore
metadata:
  name: rancher-restore-test
spec:
  backupFilename: rancher-daily-2026-03-01.tar.gz
  storageLocation:
    name: s3-backup
EOF

# 3. Verify cluster configurations are present
# 4. Verify downstream clusters reconnect

# Test Velero restore
velero restore create --from-backup daily-production-20260301000000 \
  --include-namespaces production \
  --namespace-mappings production:production-restore \
  --wait
```

## Backup Checklist

- Rancher management plane: daily backup to S3, 14-day retention
- etcd: every 6 hours, uploaded to S3
- Application data: Velero daily backup, 30-day retention
- Databases: application-consistent backups hourly or daily
- Backups encrypted at rest
- Restore procedures documented and tested quarterly
- Backup success/failure alerts configured

## Conclusion

A complete Rancher backup strategy requires all three layers: management plane, etcd, and application data. The Rancher Backup Operator handles the management plane, RKE2's built-in snapshot handles etcd, and Velero handles application workloads. Test restores quarterly-untested backups are not backups. Store backups in a separate AWS account or geographic region to protect against account compromise or regional failures.
