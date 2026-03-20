# How to Back Up Harvester Configuration

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Harvester, Backup, Configuration, Kubernetes, etcd, Disaster Recovery, SUSE Rancher

Description: Learn how to back up Harvester cluster configuration including etcd snapshots, VM configurations, network settings, and Longhorn storage data for comprehensive disaster recovery coverage.

---

Backing up Harvester requires capturing both the Kubernetes control plane state (etcd) and the Longhorn storage data. This guide covers both layers for complete disaster recovery coverage.

---

## What to Back Up

| Component | Method | Frequency |
|---|---|---|
| etcd (cluster state) | RKE2 etcd snapshot | Daily |
| VM configurations | kubectl export + git | On change |
| Longhorn volume data | Longhorn backups to S3 | Hourly/Daily |
| Harvester settings | kubectl export | Weekly |
| Network configs | YAML export | On change |

---

## Step 1: Back Up etcd (RKE2)

Harvester runs on RKE2. Use RKE2's built-in etcd snapshot feature:

```bash
# On a Harvester control plane node
# Take an on-demand etcd snapshot
rke2 etcd-snapshot save \
  --name harvester-config-$(date +%Y%m%d%H%M%S)

# Configure automatic etcd snapshots to S3
# /etc/rancher/rke2/config.yaml
etcd-snapshot-schedule-cron: "0 2 * * *"   # Daily at 2 AM
etcd-snapshot-retention: 7
etcd-s3: true
etcd-s3-bucket: my-harvester-backups
etcd-s3-region: us-east-1
etcd-s3-access-key: <key>
etcd-s3-secret-key: <secret>

# List available snapshots
rke2 etcd-snapshot ls
```

---

## Step 2: Export VM Configurations

```bash
# Export all VM definitions
kubectl get vm -A -o yaml > vm-configs-$(date +%Y%m%d).yaml

# Export network attachment definitions
kubectl get nad -A -o yaml > network-configs-$(date +%Y%m%d).yaml

# Export VM images list
kubectl get virtualmachineimage -A -o yaml > vm-images-$(date +%Y%m%d).yaml

# Export storage classes
kubectl get storageclass -o yaml > storageclasses-$(date +%Y%m%d).yaml
```

---

## Step 3: Configure Longhorn Backup Target

```bash
# Create S3 credentials secret
kubectl create secret generic harvester-backup-s3 \
  -n longhorn-system \
  --from-literal=AWS_ACCESS_KEY_ID=<key> \
  --from-literal=AWS_SECRET_ACCESS_KEY=<secret>

# Configure the backup target in Longhorn settings
kubectl patch setting.longhorn.io backup-target \
  -n longhorn-system --type merge \
  -p '{"value":"s3://harvester-vm-backups@us-east-1/"}'

kubectl patch setting.longhorn.io backup-target-credential-secret \
  -n longhorn-system --type merge \
  -p '{"value":"harvester-backup-s3"}'
```

---

## Step 4: Configure VM Backup Schedule

In Harvester UI, go to **Virtual Machines > [VM Name] > Schedule Backup**. Or create Longhorn recurring jobs:

```yaml
# recurring-backup-vms.yaml
apiVersion: longhorn.io/v1beta2
kind: RecurringJob
metadata:
  name: daily-vm-backup
  namespace: longhorn-system
spec:
  cron: "0 3 * * *"
  task: backup
  groups:
    - vm-volumes
  retain: 7
  concurrency: 2
```

---

## Step 5: Automate Backup Verification

```bash
#!/bin/bash
# backup-verify.sh — run weekly to test restores
set -e

# List recent backups
kubectl get lhbackup -n longhorn-system \
  --sort-by=.metadata.creationTimestamp \
  | tail -10

# Check backup status — all should show "Completed"
FAILED=$(kubectl get lhbackup -n longhorn-system \
  -o jsonpath='{.items[?(@.status.state!="Completed")].metadata.name}')

if [ -n "$FAILED" ]; then
  echo "FAILED BACKUPS: $FAILED"
  exit 1
fi
echo "All backups verified"
```

---

## Best Practices

- Test disaster recovery quarterly by restoring to a test Harvester cluster.
- Store etcd snapshots in a different location from the Harvester cluster (S3, remote NFS).
- Keep VM configuration YAML files in Git so you have a version history of VM changes.
- Document your recovery procedure — the person performing recovery may not be the person who set up the backup.
