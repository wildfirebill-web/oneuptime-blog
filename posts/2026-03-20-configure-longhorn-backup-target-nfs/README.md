# How to Configure Longhorn Backup Target to NFS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Longhorn, Kubernetes, Storage, Backup, NFS, Configuration

Description: Learn how to configure an NFS server as a Longhorn backup target for storing volume backups on-premises without requiring cloud storage.

## Introduction

NFS (Network File System) is a popular choice for on-premises Longhorn backup targets. It allows you to use existing NAS devices or NFS servers as backup destinations, which is particularly useful in environments without internet access or where data sovereignty requirements prevent using cloud storage. This guide walks through setting up both the NFS server and configuring Longhorn to use it.

## Prerequisites

- Longhorn installed on your Kubernetes cluster
- An NFS server accessible from all cluster nodes
- `nfs-common` (Debian/Ubuntu) or `nfs-utils` (RHEL/CentOS) installed on each Kubernetes node
- Network connectivity between cluster nodes and the NFS server on port 2049

## Install NFS Client on All Nodes

```bash
# Ubuntu/Debian

apt-get install -y nfs-common

# RHEL/CentOS/Rocky Linux
yum install -y nfs-utils
systemctl enable --now nfs-client.target
```

## Set Up an NFS Server (Optional)

If you do not have an existing NFS server, here is how to set one up on a dedicated server:

```bash
# Install NFS server on Ubuntu/Debian
apt-get install -y nfs-kernel-server

# Create the export directory
mkdir -p /export/longhorn-backups
chmod 777 /export/longhorn-backups

# Add the export to /etc/exports
# Replace 10.0.0.0/24 with your cluster's subnet
echo '/export/longhorn-backups 10.0.0.0/24(rw,sync,no_subtree_check,no_root_squash)' \
  >> /etc/exports

# Export the shares
exportfs -ar

# Start and enable the NFS server
systemctl enable --now nfs-kernel-server

# Verify the export
showmount -e localhost
```

## Verify NFS Connectivity from Cluster Nodes

Before configuring Longhorn, verify that the NFS share is accessible from your nodes:

```bash
# Test NFS mount from a worker node
# Replace 192.168.1.100 with your NFS server IP
showmount -e 192.168.1.100

# Test mounting the share
mkdir -p /mnt/nfs-test
mount -t nfs 192.168.1.100:/export/longhorn-backups /mnt/nfs-test
ls /mnt/nfs-test

# Unmount after test
umount /mnt/nfs-test
```

## Configure Longhorn Backup Target

### Via kubectl

```bash
# Set the NFS backup target
# Format: nfs://server-ip-or-hostname:/path/to/export
kubectl patch settings.longhorn.io backup-target \
  -n longhorn-system \
  --type merge \
  -p '{"value": "nfs://192.168.1.100:/export/longhorn-backups"}'

# NFS does not require credentials, so clear the secret setting
kubectl patch settings.longhorn.io backup-target-credential-secret \
  -n longhorn-system \
  --type merge \
  -p '{"value": ""}'
```

### Via Longhorn UI

1. Navigate to **Setting** → **General**
2. Find **Backup Target**
3. Enter the NFS URL: `nfs://192.168.1.100:/export/longhorn-backups`
4. Leave **Backup Target Credential Secret** empty
5. Click **Save**

## Verify the Connection

```bash
# Check the backup target setting
kubectl get settings.longhorn.io backup-target \
  -n longhorn-system -o yaml

# Check backup volumes - should list backups if connection works
kubectl get backupvolumes.longhorn.io -n longhorn-system
```

In the Longhorn UI, navigate to **Backup** and verify no connection errors appear.

## Configure Backup Subdirectory Structure

Longhorn automatically creates a directory structure on the NFS share:

```text
/export/longhorn-backups/
  backupstore/
    volumes/
      <volume-name>/
        volume.cfg
        backups/
          <backup-id>/
            ...
```

## Create a Test Backup

```bash
# Trigger a manual backup via the Longhorn UI or kubectl
# In the UI: Volume → three-dot menu → Create Backup

# Verify the backup appeared on the NFS server (from the NFS server)
ls /export/longhorn-backups/backupstore/volumes/
```

## Configure Recurring Backups to NFS

```yaml
# recurring-nfs-backup.yaml - Automated daily backup to NFS target
apiVersion: longhorn.io/v1beta2
kind: RecurringJob
metadata:
  name: daily-nfs-backup
  namespace: longhorn-system
spec:
  cron: "0 2 * * *"     # Every day at 2 AM
  task: "backup"         # Backup to the configured NFS target
  retain: 30             # Keep 30 daily backups (~1 month)
  concurrency: 2
  labels:
    schedule: daily
```

```bash
kubectl apply -f recurring-nfs-backup.yaml

# Associate with volumes
kubectl label volumes.longhorn.io <volume-name> \
  -n longhorn-system \
  "recurring-job.longhorn.io/daily-nfs-backup=enabled"
```

## NFS Server Sizing Considerations

When sizing your NFS server for Longhorn backups:

```bash
# Estimate backup storage needed:
# Total PVC data × (1 + incremental-factor) × retention-count

# Example: 500 GB of data, 30 days retention, ~10% daily change rate
# Initial full backup: 500 GB
# Daily incrementals: ~50 GB × 30 = 1,500 GB
# Total: ~2 TB needed

# Check current backup storage usage on the NFS server
du -sh /export/longhorn-backups/
```

## Troubleshooting NFS Backup Issues

### Check NFS Mount

```bash
# Verify the NFS mount within a Longhorn manager pod
kubectl exec -it -n longhorn-system \
  $(kubectl get pod -n longhorn-system -l app=longhorn-manager -o name | head -1) \
  -- mount | grep nfs
```

### Check Longhorn Manager Logs

```bash
# Look for NFS-related errors
kubectl logs -n longhorn-system \
  -l app=longhorn-manager \
  --tail=200 | grep -i "nfs\|backup"
```

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Permission denied | NFS export permissions | Ensure `no_root_squash` is set |
| Connection timeout | Firewall blocking port 2049 | Open TCP/UDP port 2049 |
| Mount fails | nfs-common not installed | Install NFS client on all nodes |

## Conclusion

Configuring Longhorn with an NFS backup target is an excellent option for on-premises deployments that require local backup storage without cloud dependencies. NFS is widely supported, cost-effective, and familiar to most operations teams. Once configured, Longhorn's recurring backup system automates the backup process, ensuring consistent data protection for your Kubernetes volumes.
