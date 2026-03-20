# How to Configure Persistent Storage for Databases in Rancher - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Storage, PersistentVolume, Database, Longhorn

Description: Configure persistent storage for databases in Rancher using StorageClasses, PersistentVolumes, and Longhorn for reliable data persistence across pod restarts.

## Introduction

Databases require persistent storage that survives pod restarts, rescheduling, and node failures. Rancher provides multiple storage options: built-in cloud provider storage classes, local storage, and Longhorn (Rancher's own storage solution). This guide covers configuring storage for database workloads with the right performance and reliability characteristics.

## Prerequisites

- Rancher-managed cluster
- kubectl with cluster-admin access
- Longhorn or a cloud storage provider configured
- Helm 3.x

## Step 1: Install Longhorn for Persistent Storage

Longhorn is Rancher's recommended distributed block storage:

```bash
# Install Longhorn from Rancher Apps catalog

# Or use Helm
helm repo add longhorn https://charts.longhorn.io
helm repo update

helm install longhorn longhorn/longhorn \
  --namespace longhorn-system \
  --create-namespace \
  --set defaultSettings.defaultReplicaCount=3 \
  --set defaultSettings.backupTarget=s3://my-backups@us-east-1/ \
  --wait

# Verify Longhorn is running
kubectl get pods -n longhorn-system
```

## Step 2: Create StorageClasses for Different Performance Tiers

```yaml
# storage-classes.yaml - Different storage tiers for databases
# High-performance SSD storage for write-intensive databases
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: database-ssd
  annotations:
    storageclass.kubernetes.io/is-default-class: "false"
provisioner: driver.longhorn.io
allowVolumeExpansion: true
reclaimPolicy: Retain
volumeBindingMode: Immediate
parameters:
  # Number of replicas for data safety
  numberOfReplicas: "3"
  # Stale replica timeout
  staleReplicaTimeout: "2880"
  # Data locality - prefer pods and volumes on same node
  dataLocality: best-effort
  # Disk selector for SSD nodes
  diskSelector: ssd
  # Encrypt the volume
  encrypted: "false"
---
# Standard storage for less demanding workloads
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: database-standard
provisioner: driver.longhorn.io
allowVolumeExpansion: true
reclaimPolicy: Retain
parameters:
  numberOfReplicas: "2"
  staleReplicaTimeout: "2880"
---
# Fast, ephemeral storage for temp data (no replication)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: database-fast-local
provisioner: rancher.io/local-path
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

## Step 3: Create PersistentVolumeClaims for Databases

```yaml
# mysql-pvcs.yaml - PVCs for MySQL deployment
# Data volume
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-data
  namespace: databases
  annotations:
    # Longhorn backup schedule
    backup.longhorn.io/backup-schedule: "0 2 * * *"
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: database-ssd
  resources:
    requests:
      storage: 50Gi
---
# Binlog volume
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-binlog
  namespace: databases
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: database-standard
  resources:
    requests:
      storage: 20Gi
---
# Backup volume
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-backup
  namespace: databases
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: database-standard
  resources:
    requests:
      storage: 100Gi
```

## Step 4: Configure StatefulSet with Volumed Storage

```yaml
# mysql-statefulset.yaml - MySQL with proper storage configuration
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  namespace: databases
spec:
  serviceName: mysql
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:8.0
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: root-password
          ports:
            - containerPort: 3306
          volumeMounts:
            # Main data directory
            - name: data
              mountPath: /var/lib/mysql
            # Separate volume for binary logs
            - name: binlog
              mountPath: /var/lib/mysql-binlog
          # Tune filesystem settings for MySQL
          lifecycle:
            postStart:
              exec:
                command:
                  - /bin/bash
                  - -c
                  - "echo never > /sys/kernel/mm/transparent_hugepage/enabled || true"
      # Tune kernel settings for database performance
      initContainers:
        - name: tune-system
          image: busybox
          securityContext:
            privileged: true
          command:
            - /bin/sh
            - -c
            - |
              # Increase vm.swappiness for database workloads
              sysctl -w vm.swappiness=10
              # Increase dirty ratio
              sysctl -w vm.dirty_ratio=15
              sysctl -w vm.dirty_background_ratio=5
  volumeClaimTemplates:
    # Automatically provision PVCs for each pod
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: database-ssd
        resources:
          requests:
            storage: 50Gi
    - metadata:
        name: binlog
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: database-standard
        resources:
          requests:
            storage: 20Gi
```

## Step 5: Resize Persistent Volumes

```bash
# Resize a PVC (requires StorageClass with allowVolumeExpansion: true)
kubectl patch pvc mysql-data \
  -n databases \
  --type merge \
  -p '{"spec": {"resources": {"requests": {"storage": "100Gi"}}}}'

# Watch the resize operation
kubectl get pvc mysql-data -n databases -w

# Verify the filesystem was resized
kubectl exec -n databases mysql-0 -- df -h /var/lib/mysql
```

## Step 6: Configure Longhorn Backup for Database Volumes

```yaml
# longhorn-backup-schedule.yaml - Longhorn recurring job for database backups
apiVersion: longhorn.io/v1beta2
kind: RecurringJob
metadata:
  name: database-backup
  namespace: longhorn-system
spec:
  # Backup to S3 daily
  cron: "0 2 * * *"
  task: backup
  groups:
    - databases  # Apply to volumes with this group label
  retain: 14    # Keep 14 backups
  concurrency: 2
  labels:
    backup-type: scheduled
```

Tag database volumes with the backup group:

```bash
# Apply backup label to PVC via annotation
kubectl annotate pvc mysql-data \
  -n databases \
  "recurring-job-group.longhorn.io/databases=enabled"
```

## Step 7: Monitor Storage Usage

```bash
# Check PVC usage across namespaces
kubectl get pvc --all-namespaces -o json | \
  jq -r '.items[] | select(.metadata.namespace | contains("database")) | "\(.metadata.namespace)/\(.metadata.name): \(.spec.resources.requests.storage)"'

# Longhorn volume details
kubectl get volume -n longhorn-system

# Check node disk usage
kubectl get node -o json | \
  jq -r '.items[] | "\(.metadata.name): \(.status.allocatable.storage)"'
```

## Step 8: Database Storage Best Practices

```yaml
# storage-best-practices-notes.yaml - Documentation as config
# Database storage recommendations:
#
# MySQL/MariaDB:
#   - Separate volumes for data, binlogs, and backup
#   - Use ReadWriteOnce access mode
#   - Enable WAL archiving
#
# PostgreSQL:
#   - Main data: high IOPS SSD storage
#   - WAL logs: separate volume
#   - Base backups: standard storage
#
# MongoDB:
#   - Data: SSD with high IOPS
#   - Journals: same volume as data
#   - Backups: standard/cold storage
#
# Redis:
#   - RDB + AOF: same volume acceptable
#   - Small, fast storage (redis is memory-based)
#
# Elasticsearch:
#   - Hot nodes: NVMe SSD
#   - Warm nodes: SSD
#   - Cold nodes: HDD acceptable
```

## Conclusion

Proper persistent storage configuration is critical for reliable database deployments on Rancher. Longhorn provides a Kubernetes-native distributed block storage solution that integrates seamlessly with Rancher's management plane. Always use separate volumes for different database components (data, logs, backups), configure appropriate StorageClasses with retention policies, and set up automated backups to external object storage. Monitor disk usage and set up alerts before volumes fill up to prevent data loss.
