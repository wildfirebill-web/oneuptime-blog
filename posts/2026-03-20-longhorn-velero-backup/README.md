# How to Use Longhorn with Velero for Backup

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Longhorn, Kubernetes, Storage, Velero, Backup, Disaster Recovery

Description: Configure Velero to use Longhorn's CSI snapshot capabilities for application-consistent Kubernetes backups including both application state and persistent volume data.

## Introduction

Velero is a popular open-source tool for backing up and restoring Kubernetes cluster resources and persistent volumes. When combined with Longhorn's CSI snapshot support, Velero can create application-consistent backups that capture both Kubernetes resource definitions and the associated volume data. This guide covers the complete setup and usage of Velero with Longhorn.

## How Velero + Longhorn Works

1. Velero triggers a Kubernetes VolumeSnapshot via the CSI plugin
2. Longhorn creates a local snapshot of the volume
3. Velero optionally uploads the snapshot to object storage
4. For restore, Velero creates new PVCs from the VolumeSnapshot data

## Prerequisites

- Longhorn installed with CSI Snapshotter configured
- An S3-compatible object storage bucket for Velero metadata
- `kubectl` and `velero` CLI installed

## Step 1: Install the Velero CLI

```bash
# Download Velero CLI (Linux)
VERSION="v1.13.0"
curl -LO https://github.com/vmware-tanzu/velero/releases/download/${VERSION}/velero-${VERSION}-linux-amd64.tar.gz
tar -xvf velero-${VERSION}-linux-amd64.tar.gz
mv velero-${VERSION}-linux-amd64/velero /usr/local/bin/
velero version --client-only
```

## Step 2: Create S3 Bucket and Credentials for Velero

```bash
# Create S3 bucket for Velero
aws s3 mb s3://velero-cluster-backups --region us-east-1

# Create Velero credentials file
cat << 'EOF' > velero-credentials
[default]
aws_access_key_id=AKIAIOSFODNN7EXAMPLE
aws_secret_access_key=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
EOF
```

## Step 3: Install Velero with CSI Plugin

```bash
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.9.0,velero/velero-plugin-for-csi:v0.7.0 \
  --bucket velero-cluster-backups \
  --backup-location-config region=us-east-1 \
  --snapshot-location-config region=us-east-1 \
  --credentials-file ./velero-credentials \
  --features=EnableCSI \
  --use-node-agent

# Verify Velero is running
kubectl get pods -n velero
velero version
```

## Step 4: Configure Longhorn VolumeSnapshotClass for Velero

Velero looks for a VolumeSnapshotClass with a specific label:

```yaml
# longhorn-velero-snapshotclass.yaml - VolumeSnapshotClass for Velero
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: longhorn-snap-class
  labels:
    # Velero uses this label to find the right VolumeSnapshotClass
    velero.io/csi-volumesnapshot-class: "true"
driver: driver.longhorn.io
deletionPolicy: Retain  # Retain snapshots during Velero backup lifecycle
```

```bash
kubectl apply -f longhorn-velero-snapshotclass.yaml
```

## Step 5: Create a Test Application

```yaml
# test-app.yaml - Application with Longhorn PVC for testing backup/restore
apiVersion: v1
kind: Namespace
metadata:
  name: test-app
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data
  namespace: test-app
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn
  resources:
    requests:
      storage: 5Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app
  namespace: test-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:stable
          volumeMounts:
            - name: data
              mountPath: /data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: app-data
```

```bash
kubectl apply -f test-app.yaml

# Write test data
kubectl exec -n test-app \
  $(kubectl get pod -n test-app -l app=nginx -o name) \
  -- sh -c "echo 'test backup data' > /data/test.txt"
```

## Step 6: Create a Velero Backup

```bash
# Create a backup of the test-app namespace
velero backup create test-app-backup \
  --include-namespaces test-app \
  --snapshot-volumes \
  --wait

# Check backup status
velero backup describe test-app-backup
velero backup logs test-app-backup
```

## Step 7: Simulate Disaster and Restore

```bash
# Delete the application and its data
kubectl delete namespace test-app

# Verify it is gone
kubectl get ns test-app  # Should return Not Found

# Restore from the Velero backup
velero restore create test-app-restore \
  --from-backup test-app-backup \
  --wait

# Verify restoration
kubectl get all -n test-app
kubectl get pvc -n test-app
kubectl exec -n test-app \
  $(kubectl get pod -n test-app -l app=nginx -o name) \
  -- cat /data/test.txt
# Should output: test backup data
```

## Configuring Scheduled Velero Backups

```bash
# Create a daily backup schedule for all namespaces
velero schedule create daily-backup \
  --schedule="0 2 * * *" \
  --include-namespaces "*" \
  --snapshot-volumes \
  --ttl 720h    # Keep backups for 30 days

# Create a backup schedule for specific namespaces
velero schedule create production-backup \
  --schedule="0 1 * * *" \
  --include-namespaces production,databases \
  --snapshot-volumes \
  --ttl 2160h   # Keep for 90 days

# List schedules
velero schedule get
```

## Backup Hooks for Application Consistency

For databases, use backup hooks to quiesce the application before snapshotting:

```yaml
# postgresql-with-hooks.yaml - Pre/post backup hooks for PostgreSQL
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgresql
  namespace: databases
  annotations:
    # Pre-backup hook: checkpoint the database
    pre.hook.backup.velero.io/command: '["/bin/bash", "-c", "psql -U postgres -c \"CHECKPOINT;\""]'
    pre.hook.backup.velero.io/timeout: "60s"
    # Post-backup hook: resume normal operation
    post.hook.backup.velero.io/command: '["/bin/bash", "-c", "echo backup complete"]'
```

## Monitoring Velero Backups

```bash
# Check backup status
velero backup get

# Watch for backup completion
velero backup describe test-app-backup --details

# Check for failed backups
velero backup get | grep -i fail

# Get Velero metrics (if Prometheus is configured)
kubectl port-forward -n velero svc/velero 8085:8085
curl http://localhost:8085/metrics | grep velero_backup
```

## Conclusion

Velero combined with Longhorn's CSI snapshot capabilities creates a powerful, application-aware backup and restore solution for Kubernetes. The integration provides both cluster-level resource backups (YAML definitions) and volume data backups in a single operation, enabling complete disaster recovery for stateful applications. Regular backup testing through restore drills is essential to ensure your backup strategy works when it is needed most.
