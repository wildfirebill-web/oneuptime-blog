# How to Restore VMs from Backups in Harvester

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Harvester, Kubernetes, Virtualization, HCI, Backup, Restore, Disaster Recovery

Description: Learn how to restore virtual machines from backups in Harvester, including restoring to the same cluster, a new VM, or a different Harvester cluster.

## Introduction

Restoring VMs from backups is a critical disaster recovery capability. Harvester supports restoring VMs from backups stored in S3-compatible object storage or NFS. You can restore a VM in-place (overwriting the existing VM), to a new VM on the same cluster, or import a backup into a completely different Harvester cluster — enabling cross-cluster migration and disaster recovery.

## Restore Scenarios

| Scenario | Description | Requirement |
|---|---|---|
| In-place restore | Overwrite existing VM with backup state | VM must be stopped |
| New VM restore | Create a new VM from a backup | Any state |
| Cross-cluster restore | Import backup into a different cluster | Same backup target configured |

## Prerequisites

- Harvester cluster with the backup target configured
- An existing `VirtualMachineBackup` in `Complete` state
- For cross-cluster restore: the same backup target configured on the target cluster

## Step 1: List Available Backups

### Via the UI

Navigate to **VM Backups** (under the **Backup & Snapshot** section) to see all backups with their status and sizes.

### Via kubectl

```bash
# List all VM backups
kubectl get virtualmachinebackup -n default

# Get details about a specific backup
kubectl describe virtualmachinebackup ubuntu-web-01-backup-20240315 -n default

# Check the backup is ready for restore
kubectl get virtualmachinebackup ubuntu-web-01-backup-20240315 -n default \
    -o jsonpath='{.status.phase}'
# Expected: Complete

kubectl get virtualmachinebackup ubuntu-web-01-backup-20240315 -n default \
    -o jsonpath='{.status.readyToUse}'
# Expected: true
```

## Step 2: Restore to the Same VM (In-Place Restore)

This replaces the VM's current disks with the backup data:

### Via the UI

1. Navigate to **VM Backups**
2. Find the backup you want to restore
3. Click the **⋮** menu → **Restore**
4. Select **Replace Existing VM**
5. Confirm the restore operation
6. The VM will be stopped, restored, and optionally restarted

### Via kubectl

```yaml
# vm-restore-inplace.yaml
# Restore a VM to its backup state

apiVersion: harvesterhci.io/v1beta1
kind: VirtualMachineRestore
metadata:
  name: ubuntu-web-01-restore-20240315
  namespace: default
spec:
  # Type: restore (from backup) or replaceVolumes
  type: restore
  # Source backup to restore from
  virtualMachineBackupName: ubuntu-web-01-backup-20240315
  virtualMachineBackupNamespace: default
  # Target VM to restore (must be stopped)
  target:
    apiGroup: kubevirt.io
    kind: VirtualMachine
    name: ubuntu-web-01
  # Do not keep the original volumes (delete replaced volumes)
  deletionPolicy: delete-volumes
```

```bash
# First, stop the VM
kubectl patch vm ubuntu-web-01 -n default \
    --type merge \
    -p '{"spec":{"running":false}}'

# Wait for VM to stop
echo "Waiting for VM to stop..."
kubectl wait vmi/ubuntu-web-01 -n default \
    --for delete --timeout=120s

echo "VM stopped. Initiating restore..."

# Apply the restore
kubectl apply -f vm-restore-inplace.yaml

# Watch the restore progress
kubectl get virtualmachinerestore ubuntu-web-01-restore-20240315 -n default -w

# Check restore status
kubectl describe virtualmachinerestore ubuntu-web-01-restore-20240315 -n default
```

## Step 3: Restore to a New VM

Create a new VM from a backup without affecting the original:

```yaml
# vm-restore-new.yaml
# Create a new VM from an existing backup

apiVersion: harvesterhci.io/v1beta1
kind: VirtualMachineRestore
metadata:
  name: ubuntu-web-01-dr-restore
  namespace: default
spec:
  type: restore
  virtualMachineBackupName: ubuntu-web-01-backup-20240315
  virtualMachineBackupNamespace: default
  target:
    apiGroup: kubevirt.io
    kind: VirtualMachine
    # NEW name - VM will be created fresh
    name: ubuntu-web-01-restored
  # Keep the original volumes if they exist (for new VM, there are none)
  deletionPolicy: retain-volumes
  # Custom volume names for the new VM
  newVolumes:
    - volumeBackupName: ubuntu-web-01-root
      # New PVC name for the restored disk
      volumeName: ubuntu-web-01-restored-root
```

```bash
kubectl apply -f vm-restore-new.yaml

# Monitor the restore
kubectl get virtualmachinerestore ubuntu-web-01-dr-restore -n default -w

# Once complete, start the new VM
kubectl patch vm ubuntu-web-01-restored -n default \
    --type merge \
    -p '{"spec":{"running":true}}'

# Verify the new VM started
kubectl get vmi ubuntu-web-01-restored -n default
```

## Step 4: Cross-Cluster Restore

To restore a VM on a different Harvester cluster:

### On the Target Cluster

Configure the same backup target as the source cluster:

```yaml
# backup-target-config.yaml
# Configure the same S3 bucket on the target cluster

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
# On the target cluster, apply the backup target configuration
kubectl apply -f backup-target-config.yaml

# Wait for the backup controller to scan and discover backups from the source cluster
# This may take a few minutes
kubectl get virtualmachinebackup -n default

# Once the backups appear, create the restore
kubectl apply -f - <<EOF
apiVersion: harvesterhci.io/v1beta1
kind: VirtualMachineRestore
metadata:
  name: cross-cluster-restore
  namespace: default
spec:
  type: restore
  virtualMachineBackupName: ubuntu-web-01-backup-20240315
  virtualMachineBackupNamespace: default
  target:
    apiGroup: kubevirt.io
    kind: VirtualMachine
    name: ubuntu-web-01-imported
EOF
```

## Step 5: Validate the Restored VM

After restore, always verify the VM is functioning correctly:

```bash
# Start the restored VM
kubectl patch vm ubuntu-web-01-restored -n default \
    --type merge \
    -p '{"spec":{"running":true}}'

# Wait for the VMI to be running
kubectl wait vmi/ubuntu-web-01-restored -n default \
    --for=condition=Ready=True \
    --timeout=300s

# Access the VM console to verify
virtctl console ubuntu-web-01-restored -n default

# Inside the VM, check:
# 1. Hostname and network configuration
hostname
ip addr show

# 2. Application services are running
systemctl status nginx
systemctl status postgresql

# 3. Data integrity
ls /var/www/html/
ls /var/lib/postgresql/data/
```

## Automate Disaster Recovery Testing

Regularly test restores with an automated DR test:

```bash
#!/bin/bash
# dr-test.sh - Automated restore test

BACKUP_NAME="ubuntu-web-01-backup-latest"
TEST_VM_NAME="ubuntu-web-01-dr-test"
NAMESPACE="default"

echo "=== DR Test Started at $(date) ==="

# Create test restore
kubectl apply -f - <<EOF
apiVersion: harvesterhci.io/v1beta1
kind: VirtualMachineRestore
metadata:
  name: dr-test-restore
  namespace: ${NAMESPACE}
spec:
  type: restore
  virtualMachineBackupName: ${BACKUP_NAME}
  virtualMachineBackupNamespace: ${NAMESPACE}
  target:
    apiGroup: kubevirt.io
    kind: VirtualMachine
    name: ${TEST_VM_NAME}
EOF

# Wait for restore
kubectl wait virtualmachinerestore/dr-test-restore \
    -n ${NAMESPACE} \
    --for=condition=Ready=True \
    --timeout=600s

# Start the VM
kubectl patch vm ${TEST_VM_NAME} -n ${NAMESPACE} \
    --type merge \
    -p '{"spec":{"running":true}}'

# Wait for VM
kubectl wait vmi/${TEST_VM_NAME} -n ${NAMESPACE} \
    --for=condition=Ready=True \
    --timeout=300s

echo "=== DR Test VM is running. Validating... ==="

# Run validation
VM_IP=$(kubectl get vmi ${TEST_VM_NAME} -n ${NAMESPACE} \
    -o jsonpath='{.status.interfaces[0].ipAddress}')

if curl -sf "http://${VM_IP}/healthz" > /dev/null; then
    echo "=== PASS: Application health check succeeded ==="
else
    echo "=== FAIL: Application health check failed ==="
fi

# Cleanup test VM
echo "Cleaning up test VM..."
kubectl patch vm ${TEST_VM_NAME} -n ${NAMESPACE} \
    --type merge -p '{"spec":{"running":false}}'
kubectl delete vm ${TEST_VM_NAME} -n ${NAMESPACE}
kubectl delete virtualmachinerestore dr-test-restore -n ${NAMESPACE}

echo "=== DR Test Complete ==="
```

## Conclusion

Restoring VMs from backups in Harvester is a straightforward process that supports multiple recovery scenarios from simple in-place restores to full cross-cluster disaster recovery. The Kubernetes-native API makes it easy to automate restore testing, which is a critical practice often overlooked in disaster recovery planning. Test your restore procedures regularly — ideally monthly — to ensure your backup strategy actually works when you need it most.
