# How to Back Up VMs in Harvester

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Harvester, Kubernetes, Virtualization, HCI, Backup, S3, NFS

Description: A complete guide to configuring and executing VM backups in Harvester using S3-compatible object storage or NFS as the backup target.

## Introduction

VM backups in Harvester store complete copies of VM disk data to an external target — either an S3-compatible object store or an NFS share. Unlike snapshots (which stay on the cluster), backups provide off-cluster data protection that survives hardware failures, data corruption, or complete cluster loss. This guide covers configuring the backup target and creating both manual and scheduled backups.

## Backup Architecture

```mermaid
graph LR
    VM["VM Disks\n(Longhorn)"] --> BackupController["Harvester\nBackup Controller"]
    BackupController --> S3["S3 Object Store\n(AWS, MinIO, etc.)"]
    BackupController --> NFS["NFS Share"]
```

Backups are incremental by default — only changed blocks are transferred after the initial full backup, making subsequent backups fast and storage-efficient.

## Step 1: Configure the Backup Target

### Using S3

Navigate to **Settings** → **Backup Target** in the Harvester UI:

```
Type:                S3
Endpoint:            https://s3.amazonaws.com  (or your MinIO/Ceph endpoint)
Bucket Name:         harvester-vm-backups
Region:              us-east-1
Access Key ID:       AKIAIOSFODNN7EXAMPLE
Secret Access Key:   wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
Certificate:         (leave blank for public S3; add cert for private endpoints)
Virtual Hosted-Style: true  (for AWS S3)
```

### Using NFS

```
Type:           NFS
Server Address: 192.168.1.50
Mount Point:    /exports/harvester-backups
```

### Via kubectl

```yaml
# backup-target-s3.yaml
# Configure S3 as the backup target

apiVersion: v1
kind: Secret
metadata:
  name: backup-target-secret
  namespace: harvester-system
type: Opaque
stringData:
  AWS_ACCESS_KEY_ID: "AKIAIOSFODNN7EXAMPLE"
  AWS_SECRET_ACCESS_KEY: "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
---
apiVersion: harvesterhci.io/v1beta1
kind: Setting
metadata:
  name: backup-target
  namespace: harvester-system
spec:
  value: |
    {
      "type": "s3",
      "endpoint": "https://s3.amazonaws.com",
      "bucketName": "harvester-vm-backups",
      "bucketRegion": "us-east-1",
      "secret": "backup-target-secret"
    }
```

```bash
kubectl apply -f backup-target-s3.yaml

# Verify the backup target is reachable
kubectl get setting backup-target -n harvester-system \
    -o jsonpath='{.status.conditions}' | jq .
```

## Step 2: Create a Manual VM Backup

### Via the UI

1. Navigate to **Virtual Machines**
2. Find the VM you want to back up
3. Click the **⋮** menu → **Backup**
4. Provide a backup name
5. Click **Create**

### Via kubectl

```yaml
# vm-backup.yaml
# Create an on-demand backup of a VM

apiVersion: harvesterhci.io/v1beta1
kind: VirtualMachineBackup
metadata:
  name: ubuntu-web-01-backup-20240315
  namespace: default
spec:
  # Reference to the source VM
  source:
    apiGroup: kubevirt.io
    kind: VirtualMachine
    name: ubuntu-web-01
  # Type: backup (external) or snapshot (on-cluster)
  type: backup
```

```bash
kubectl apply -f vm-backup.yaml

# Watch backup progress
kubectl get virtualmachinebackup ubuntu-web-01-backup-20240315 -n default -w

# Check backup status details
kubectl describe virtualmachinebackup ubuntu-web-01-backup-20240315 -n default

# A successful backup shows:
# PHASE: Complete
# READY: true
```

## Step 3: Schedule Recurring Backups

Harvester doesn't have a built-in backup scheduler, but you can use Kubernetes CronJobs:

```yaml
# backup-cronjob.yaml
# Daily VM backup job running at 1:00 AM

apiVersion: v1
kind: ServiceAccount
metadata:
  name: backup-job-sa
  namespace: default
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: vm-backup-role
rules:
  - apiGroups: ["harvesterhci.io"]
    resources: ["virtualmachinebackups"]
    verbs: ["create", "get", "list", "delete"]
  - apiGroups: ["kubevirt.io"]
    resources: ["virtualmachines"]
    verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: vm-backup-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: vm-backup-role
subjects:
  - kind: ServiceAccount
    name: backup-job-sa
    namespace: default
---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: daily-vm-backup
  namespace: default
spec:
  # Run daily at 1:00 AM
  schedule: "0 1 * * *"
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: backup-job-sa
          restartPolicy: OnFailure
          containers:
            - name: backup-runner
              image: bitnami/kubectl:latest
              command:
                - /bin/sh
                - -c
                - |
                  # Get list of all VMs to backup
                  DATE=$(date +%Y%m%d)

                  for VM in $(kubectl get vm -n default -o name | sed 's|virtualmachine.kubevirt.io/||'); do
                    BACKUP_NAME="${VM}-backup-${DATE}"
                    echo "Creating backup: ${BACKUP_NAME}"

                    kubectl apply -f - <<BACKUP_EOF
                  apiVersion: harvesterhci.io/v1beta1
                  kind: VirtualMachineBackup
                  metadata:
                    name: ${BACKUP_NAME}
                    namespace: default
                    labels:
                      backup-date: "${DATE}"
                      automated: "true"
                  spec:
                    source:
                      apiGroup: kubevirt.io
                      kind: VirtualMachine
                      name: ${VM}
                    type: backup
                  BACKUP_EOF

                  done

                  # Delete backups older than 7 days
                  CUTOFF=$(date -d '7 days ago' +%Y%m%d)
                  kubectl get vmbackup -n default -l automated=true \
                    -o jsonpath='{range .items[*]}{.metadata.name} {.metadata.labels.backup-date}{"\n"}{end}' | \
                    while read NAME DATE_LABEL; do
                      if [[ "${DATE_LABEL}" < "${CUTOFF}" ]]; then
                        echo "Deleting old backup: ${NAME}"
                        kubectl delete vmbackup ${NAME} -n default
                      fi
                    done
```

```bash
kubectl apply -f backup-cronjob.yaml

# Verify the CronJob is created
kubectl get cronjob daily-vm-backup -n default

# Manually trigger a backup run to test
kubectl create job --from=cronjob/daily-vm-backup test-backup-now -n default

# Watch the job
kubectl get job test-backup-now -n default -w
```

## Step 4: Verify Backup Integrity

```bash
# List all backups
kubectl get virtualmachinebackup -n default

# Check backup health - all should show READY: true
kubectl get virtualmachinebackup -n default \
    -o custom-columns=\
'NAME:.metadata.name,PHASE:.status.phase,READY:.status.readyToUse,SIZE:.status.size'

# Verify backup contents in S3
aws s3 ls s3://harvester-vm-backups/ --recursive | head -30
```

## Conclusion

VM backups in Harvester provide essential protection against data loss from hardware failures, accidental deletions, and disasters. By configuring a reliable backup target (S3 or NFS) and scheduling regular automated backups, you create a safety net that enables recovery from virtually any failure scenario. The incremental backup mechanism keeps storage costs reasonable while the Kubernetes-native API enables integration with your existing automation and monitoring workflows. Always test your backup and restore procedures before you need them in a real emergency.
