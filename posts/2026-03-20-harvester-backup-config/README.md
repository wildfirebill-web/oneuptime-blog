# How to Export and Restore Harvester Configuration

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Harvester, Backup, Configuration, Disaster Recovery, Kubernetes, SUSE Rancher, HCI

Description: Learn how to back up Harvester cluster configuration including VM settings, network configurations, storage policies, and etcd snapshots for disaster recovery.

---

Backing up Harvester configuration protects your cluster settings, VM definitions, network configurations, and storage policies. A complete backup strategy combines etcd snapshots (cluster state), VM disk backups, and exported configuration manifests.

---

## What to Back Up

| Component | Method | Frequency |
|---|---|---|
| Cluster state (etcd) | RKE2 etcd snapshots | Every 6 hours |
| VM disk data | Longhorn volume backups | Daily |
| VM definitions | kubectl export | Before changes |
| Network config | kubectl export | Before changes |
| Harvester settings | kubectl export | Weekly |

---

## Step 1: Configure etcd Snapshots

Harvester is built on RKE2, so etcd backup configuration is identical to RKE2:

```yaml
# /etc/rancher/rke2/config.yaml (on Harvester server nodes)

etcd-snapshot-schedule-cron: "0 */6 * * *"
etcd-snapshot-retention: 10
etcd-s3: true
etcd-s3-bucket: harvester-etcd-backups
etcd-s3-region: us-west-2
etcd-s3-access-key: YOUR_ACCESS_KEY
etcd-s3-secret-key: YOUR_SECRET_KEY
```

```bash
# Verify snapshots are being created
rke2 etcd-snapshot ls

# Create a manual snapshot before making changes
rke2 etcd-snapshot save --name pre-upgrade-$(date +%Y%m%d)
```

---

## Step 2: Configure Longhorn Backup for VM Disks

```bash
# Set Longhorn backup target in Harvester
# Via Harvester UI: Advanced → Backup & Snapshot → Backup Target

# Or via kubectl
kubectl patch setting -n longhorn-system \
  backup-target \
  --type merge \
  -p '{"value":"s3://harvester-vm-backups@us-west-2/"}'

kubectl patch setting -n longhorn-system \
  backup-target-credential-secret \
  --type merge \
  -p '{"value":"longhorn-backup-s3-secret"}'
```

---

## Step 3: Create VM Backup Policies

In Harvester, VM backups are taken at the Harvester level (not just the disk):

```bash
# Take a VM backup via Harvester UI:
# Virtual Machines → select VM → Take Backup

# Or via kubectl (Harvester-specific CRD)
kubectl apply -f - <<EOF
apiVersion: harvesterhci.io/v1beta1
kind: VirtualMachineBackup
metadata:
  name: my-vm-backup-$(date +%Y%m%d)
  namespace: default
spec:
  source:
    apiGroup: kubevirt.io
    kind: VirtualMachine
    name: my-vm
EOF
```

---

## Step 4: Export VM Configuration Manifests

```bash
# Export all VM definitions
kubectl get vm -A -o yaml > vm-definitions-$(date +%Y%m%d).yaml

# Export network configurations
kubectl get networkattachmentdefinition -A -o yaml > networks-$(date +%Y%m%d).yaml

# Export storage configurations
kubectl get storageclass -o yaml > storageclasses-$(date +%Y%m%d).yaml
kubectl get pvc -A -o yaml > pvcs-$(date +%Y%m%d).yaml

# Export Harvester-specific settings
kubectl get setting -n harvester-system -o yaml > harvester-settings-$(date +%Y%m%d).yaml

# Export VM images
kubectl get virtualmachineimage -n harvester-system -o yaml > vm-images-$(date +%Y%m%d).yaml
```

---

## Step 5: Automate Configuration Backups

```bash
#!/bin/bash
# backup-harvester-config.sh

BACKUP_DIR="/backup/harvester/$(date +%Y%m%d)"
mkdir -p $BACKUP_DIR

# Export all Harvester resources
resources=(
  "virtualmachine"
  "virtualmachineimage"
  "networkattachmentdefinition"
  "storageclass"
  "clusternetwork"
  "vlanconfig"
)

for resource in "${resources[@]}"; do
  kubectl get $resource -A -o yaml > "$BACKUP_DIR/${resource}.yaml" 2>/dev/null
  echo "Backed up: $resource"
done

# Compress the backup
tar -czf "/backup/harvester-config-$(date +%Y%m%d).tar.gz" -C "$BACKUP_DIR" .
echo "Backup complete: /backup/harvester-config-$(date +%Y%m%d).tar.gz"
```

Schedule this script as a cron job:

```bash
# /etc/cron.d/harvester-backup
0 1 * * * root /usr/local/bin/backup-harvester-config.sh
```

---

## Step 6: Test Restore Procedure

Regularly test your backup by restoring to a staging Harvester instance:

```bash
# Restore VM definitions to a new Harvester cluster
kubectl apply -f vm-definitions-20260320.yaml

# Restore VM from Longhorn backup
kubectl apply -f - <<EOF
apiVersion: harvesterhci.io/v1beta1
kind: VirtualMachineRestore
metadata:
  name: restore-my-vm
  namespace: default
spec:
  target:
    apiGroup: kubevirt.io
    kind: VirtualMachine
    name: my-vm-restored
  virtualMachineBackupName: my-vm-backup-20260320
  newVM: true
EOF
```

---

## Best Practices

- Store all three backup types (etcd, Longhorn volumes, config manifests) in separate S3 buckets or locations - a corrupted backup location should not affect all backups.
- Test your full restore process quarterly - a backup is only valuable if the restore succeeds.
- Back up before every Harvester upgrade - the upgrade process modifies cluster state and a rollback requires a working etcd snapshot.
