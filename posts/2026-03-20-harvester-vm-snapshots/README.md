# How to Take VM Snapshots in Harvester

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Harvester, Kubernetes, Virtualization, HCI, Snapshots, Backup

Description: Learn how to take, manage, and restore virtual machine snapshots in Harvester for point-in-time recovery and safe change management.

## Introduction

VM snapshots in Harvester capture the state of a VM's disks at a specific point in time. Snapshots are useful before applying OS updates, configuration changes, or software upgrades — if something goes wrong, you can quickly revert to the pre-change state. Harvester uses Longhorn's snapshot technology to create space-efficient, copy-on-write snapshots stored on the same cluster storage.

## Snapshot vs. Backup

| Feature | Snapshot | Backup |
|---|---|---|
| Location | Same cluster (Longhorn) | External (S3/NFS) |
| Speed | Fast (seconds) | Slower (minutes) |
| Data protection | Limited (same hardware) | Full (off-cluster) |
| Retention | Short-term | Long-term |
| Use case | Pre-change safety net | Disaster recovery |

## Step 1: Take a Snapshot via the UI

1. Navigate to **Virtual Machines**
2. Find the VM you want to snapshot
3. Click the **⋮** (Actions) menu → **Take Snapshot**
4. Provide a snapshot name and optional description
5. Click **Create**

The snapshot appears in the **VM Snapshots** section of the VM details.

## Step 2: Take a Snapshot via kubectl

```yaml
# vm-snapshot.yaml
# Take a snapshot of a VM

apiVersion: snapshot.kubevirt.io/v1alpha1
kind: VirtualMachineSnapshot
metadata:
  name: ubuntu-web-01-before-upgrade
  namespace: default
spec:
  # Reference to the VM to snapshot
  source:
    apiGroup: kubevirt.io
    kind: VirtualMachine
    name: ubuntu-web-01
```

```bash
kubectl apply -f vm-snapshot.yaml

# Watch the snapshot creation progress
kubectl get virtualmachinesnapshot ubuntu-web-01-before-upgrade -n default -w

# A snapshot is ready when:
# READY: true
# PHASE: Succeeded
kubectl get vmsnapshot ubuntu-web-01-before-upgrade -n default \
    -o jsonpath='{.status.phase}'
```

## Step 3: List Snapshots

```bash
# List all VM snapshots
kubectl get virtualmachinesnapshot -n default

# List snapshots for a specific VM
kubectl get virtualmachinesnapshot -n default \
    --field-selector spec.source.name=ubuntu-web-01

# Get detailed snapshot information
kubectl describe virtualmachinesnapshot ubuntu-web-01-before-upgrade -n default

# Check snapshot size and contents
kubectl get virtualmachinesnapshot ubuntu-web-01-before-upgrade -n default \
    -o jsonpath='{.status}' | jq .
```

## Step 4: Restore a VM from a Snapshot

### Restore to the Same VM (In-Place Restore)

```yaml
# vm-restore-inplace.yaml
# Restore a VM to a snapshot state

apiVersion: snapshot.kubevirt.io/v1alpha1
kind: VirtualMachineRestore
metadata:
  name: ubuntu-web-01-restore
  namespace: default
spec:
  # Target VM to restore
  target:
    apiGroup: kubevirt.io
    kind: VirtualMachine
    name: ubuntu-web-01
  # Source snapshot to restore from
  virtualMachineSnapshotName: ubuntu-web-01-before-upgrade
```

```bash
# The VM must be stopped before restoring
kubectl patch vm ubuntu-web-01 -n default \
    --type merge \
    -p '{"spec":{"running":false}}'

# Wait for the VM to stop
kubectl wait vmi/ubuntu-web-01 -n default \
    --for delete --timeout=60s

# Apply the restore
kubectl apply -f vm-restore-inplace.yaml

# Watch restore progress
kubectl get virtualmachinerestore ubuntu-web-01-restore -n default -w

# Once complete, start the VM
kubectl patch vm ubuntu-web-01 -n default \
    --type merge \
    -p '{"spec":{"running":true}}'
```

### Restore to a New VM

```yaml
# vm-restore-new.yaml
# Create a new VM from a snapshot

apiVersion: snapshot.kubevirt.io/v1alpha1
kind: VirtualMachineRestore
metadata:
  name: ubuntu-web-01-clone
  namespace: default
spec:
  target:
    apiGroup: kubevirt.io
    kind: VirtualMachine
    # Name of the NEW VM to create
    name: ubuntu-web-01-clone
  virtualMachineSnapshotName: ubuntu-web-01-before-upgrade
```

```bash
kubectl apply -f vm-restore-new.yaml

# The new VM will be created with the snapshot data
kubectl get vm ubuntu-web-01-clone -n default

# Start the new VM
kubectl patch vm ubuntu-web-01-clone -n default \
    --type merge \
    -p '{"spec":{"running":true}}'
```

## Step 5: Automate Snapshots with Recurring Jobs

For regular automatic snapshots, use Longhorn recurring jobs:

```yaml
# recurring-snapshot-job.yaml
# Take daily snapshots of all VM volumes with retention of 7

apiVersion: longhorn.io/v1beta2
kind: RecurringJob
metadata:
  name: daily-vm-snapshot
  namespace: longhorn-system
spec:
  # Cron expression: daily at 2:00 AM
  cron: "0 2 * * *"
  task: snapshot
  groups:
    - default
  retain: 7     # Keep 7 snapshots
  concurrency: 2
  labels:
    origin: scheduled
```

```bash
kubectl apply -f recurring-snapshot-job.yaml

# Assign the recurring job to a specific volume
kubectl label volume.longhorn.io ubuntu-web-01-root -n longhorn-system \
    recurring-job.longhorn.io/daily-vm-snapshot=enabled
```

## Step 6: Snapshot Before OS Updates (Best Practice Workflow)

Here's an automated workflow for safe OS updates:

```bash
#!/bin/bash
# safe-update.sh - Take snapshot, update VM, verify, or rollback

VM_NAME="ubuntu-web-01"
NAMESPACE="default"
SNAPSHOT_NAME="${VM_NAME}-pre-update-$(date +%Y%m%d%H%M%S)"

echo "Step 1: Taking pre-update snapshot..."
kubectl apply -f - <<EOF
apiVersion: snapshot.kubevirt.io/v1alpha1
kind: VirtualMachineSnapshot
metadata:
  name: ${SNAPSHOT_NAME}
  namespace: ${NAMESPACE}
spec:
  source:
    apiGroup: kubevirt.io
    kind: VirtualMachine
    name: ${VM_NAME}
EOF

# Wait for snapshot to complete
kubectl wait virtualmachinesnapshot/${SNAPSHOT_NAME} \
    -n ${NAMESPACE} \
    --for=condition=Ready=True \
    --timeout=300s

echo "Snapshot ${SNAPSHOT_NAME} is ready"
echo "Now apply your updates to the VM"
echo "If updates succeed: kubectl delete vmsnapshot ${SNAPSHOT_NAME} -n ${NAMESPACE}"
echo "If updates fail: restore from ${SNAPSHOT_NAME}"
```

## Deleting Snapshots

```bash
# Delete a specific snapshot
kubectl delete virtualmachinesnapshot ubuntu-web-01-before-upgrade -n default

# Delete all snapshots older than 30 days (using labels)
kubectl get virtualmachinesnapshot -n default \
    -o json | jq -r \
    '.items[] | select(.metadata.creationTimestamp < "2024-01-01") | .metadata.name' | \
    xargs -I {} kubectl delete virtualmachinesnapshot {} -n default
```

## Conclusion

VM snapshots in Harvester provide a fast and reliable safety net for change management. The ability to take a snapshot before any risky operation — OS upgrades, configuration changes, or application deployments — means you can make changes with confidence, knowing that rollback is just minutes away. For production environments, combine on-cluster snapshots with off-cluster backups to achieve comprehensive data protection: snapshots for short-term recovery and backups for long-term disaster recovery.
