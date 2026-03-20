# How to Back Up Harvester Configuration

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Harvester, Kubernetes, Virtualization, HCI, Backup, Configuration, DR

Description: Learn how to back up Harvester cluster configuration, including Kubernetes resources, etcd data, and custom settings for disaster recovery.

## Introduction

Backing up Harvester configuration goes beyond just backing up VM data. A comprehensive configuration backup includes the Kubernetes resource definitions (VMs, networks, settings), etcd snapshots (the cluster's source of truth), and custom configurations (users, certificates, network settings). This allows you to rebuild the cluster configuration from scratch after a catastrophic failure, even on different hardware.

## What to Back Up

| Component | Location | Method |
|---|---|---|
| etcd data | In-cluster | RKE2 etcd snapshots |
| Kubernetes resources | Cluster API | kubectl export |
| Harvester settings | Cluster API | kubectl export |
| Network configs | Cluster API | kubectl export |
| VM image metadata | Cluster API | kubectl export |
| TLS certificates | Node filesystem | File backup |
| SSH keys | Cluster API | kubectl export |

## Step 1: Configure Automatic etcd Snapshots

RKE2 (the Kubernetes distribution in Harvester) supports automated etcd snapshots:

```yaml
# rke2-server-config.yaml
# Configure RKE2 etcd snapshots on each server node
# /etc/rancher/rke2/config.yaml

cluster-init: true  # Only on the first node

# etcd snapshot configuration
etcd-snapshot-schedule-cron: "0 */6 * * *"    # Every 6 hours
etcd-snapshot-retention: 10                    # Keep 10 snapshots
etcd-snapshot-dir: /var/lib/rancher/rke2/server/db/snapshots

# Optional: Store snapshots in S3
etcd-s3: true
etcd-s3-endpoint: "s3.amazonaws.com"
etcd-s3-access-key: "AKIAIOSFODNN7EXAMPLE"
etcd-s3-secret-key: "wJalrXUtnFEMI/K7MDENG"
etcd-s3-bucket: "my-harvester-etcd-backups"
etcd-s3-region: "us-east-1"
etcd-s3-folder: "harvester-etcd"
```

```bash
# Apply the configuration (requires RKE2 restart)
sudo systemctl restart rke2-server

# Manually trigger an etcd snapshot
sudo rke2 etcd-snapshot save \
    --name manual-backup-$(date +%Y%m%d%H%M%S)

# List existing snapshots
sudo rke2 etcd-snapshot ls

# Expected output:
# local/harvester-master-1704345600  (timestamp-based names)
# local/manual-backup-20240104120000
```

## Step 2: Export Kubernetes Resources

Export all Harvester-specific Kubernetes resources for documentation and recovery:

```bash
#!/bin/bash
# backup-k8s-resources.sh
# Export all Harvester Kubernetes resources

BACKUP_DIR="/backup/harvester-config-$(date +%Y%m%d)"
mkdir -p ${BACKUP_DIR}

export KUBECONFIG=/etc/rancher/rke2/rke2.yaml

echo "Exporting Kubernetes resources to ${BACKUP_DIR}..."

# ===== HARVESTER SETTINGS =====
echo "Exporting Harvester settings..."
kubectl get settings.harvesterhci.io -n harvester-system -o yaml \
    > ${BACKUP_DIR}/harvester-settings.yaml

# ===== VM IMAGES =====
echo "Exporting VM image definitions..."
kubectl get virtualmachineimages -A -o yaml \
    > ${BACKUP_DIR}/vm-images.yaml

# ===== VM TEMPLATES =====
echo "Exporting VM templates..."
kubectl get virtualmachinetemplates -A -o yaml \
    > ${BACKUP_DIR}/vm-templates.yaml

kubectl get virtualmachinetemplateversions -A -o yaml \
    >> ${BACKUP_DIR}/vm-templates.yaml

# ===== VIRTUAL MACHINES =====
echo "Exporting VM definitions..."
kubectl get virtualmachines -A -o yaml \
    > ${BACKUP_DIR}/virtual-machines.yaml

# ===== NETWORKS =====
echo "Exporting network configurations..."
kubectl get clusternetworks -o yaml \
    > ${BACKUP_DIR}/cluster-networks.yaml

kubectl get network-attachment-definitions -A -o yaml \
    > ${BACKUP_DIR}/vm-networks.yaml

kubectl get nodenetworks -n harvester-system -o yaml \
    >> ${BACKUP_DIR}/vm-networks.yaml

# ===== SSH KEYPAIRS =====
echo "Exporting SSH keypairs..."
kubectl get keypairs -A -o yaml \
    > ${BACKUP_DIR}/ssh-keypairs.yaml

# ===== NAMESPACES =====
echo "Exporting namespace configurations..."
kubectl get namespaces -o yaml \
    > ${BACKUP_DIR}/namespaces.yaml

# ===== RBAC =====
echo "Exporting RBAC configuration..."
kubectl get clusterroles -o yaml > ${BACKUP_DIR}/rbac-clusterroles.yaml
kubectl get clusterrolebindings -o yaml > ${BACKUP_DIR}/rbac-clusterrolebindings.yaml
kubectl get roles -A -o yaml > ${BACKUP_DIR}/rbac-roles.yaml
kubectl get rolebindings -A -o yaml > ${BACKUP_DIR}/rbac-rolebindings.yaml

# ===== STORAGE CLASSES =====
echo "Exporting storage classes..."
kubectl get storageclasses -o yaml > ${BACKUP_DIR}/storage-classes.yaml

# ===== CERTIFICATES =====
echo "Backing up TLS certificates..."
mkdir -p ${BACKUP_DIR}/certs

# Harvester UI certificate
kubectl get secret -n harvester-system ssl-certificates -o yaml \
    > ${BACKUP_DIR}/certs/ui-certificate.yaml 2>/dev/null || true

# ===== BACKUP CONFIGS =====
echo "Exporting backup target configuration..."
kubectl get setting backup-target -n harvester-system -o yaml \
    > ${BACKUP_DIR}/backup-target.yaml

# ===== VM BACKUPS (just metadata) =====
echo "Exporting VM backup metadata..."
kubectl get virtualmachinebackups -A -o yaml \
    > ${BACKUP_DIR}/vm-backup-metadata.yaml

# Create an archive
tar czf /backup/harvester-config-$(date +%Y%m%d).tar.gz \
    -C /backup harvester-config-$(date +%Y%m%d)/

echo "Backup complete: /backup/harvester-config-$(date +%Y%m%d).tar.gz"

# List backup size
ls -lh /backup/harvester-config-$(date +%Y%m%d).tar.gz
```

```bash
chmod +x backup-k8s-resources.sh

# Run the backup
./backup-k8s-resources.sh

# Schedule daily backups
echo "0 2 * * * /opt/scripts/backup-k8s-resources.sh >> /var/log/harvester-backup.log 2>&1" | \
    crontab -
```

## Step 3: Back Up Node Configuration Files

```bash
#!/bin/bash
# backup-node-config.sh
# Back up OS-level configuration files on each node

BACKUP_DIR="/backup/node-config-$(date +%Y%m%d)"
mkdir -p ${BACKUP_DIR}

# Files to back up on each node
CONFIG_FILES=(
    "/etc/rancher/rke2/config.yaml"
    "/etc/sysconfig/network/ifcfg-*"
    "/etc/modprobe.d/*.conf"
    "/etc/sysctl.d/*.conf"
    "/etc/systemd/system/sriov-vfs.service"
    "/etc/chrony.conf"
    "/etc/hosts"
)

for NODE in 192.168.1.11 192.168.1.12 192.168.1.13; do
    NODE_NAME=$(ssh rancher@${NODE} hostname)
    mkdir -p ${BACKUP_DIR}/${NODE_NAME}

    for FILE_PATTERN in "${CONFIG_FILES[@]}"; do
        ssh rancher@${NODE} "sudo cat ${FILE_PATTERN} 2>/dev/null" \
            > ${BACKUP_DIR}/${NODE_NAME}/$(basename ${FILE_PATTERN}) || true
    done

    # Back up RKE2 server token
    ssh rancher@${NODE} "sudo cat /var/lib/rancher/rke2/server/node-token 2>/dev/null" \
        > ${BACKUP_DIR}/${NODE_NAME}/cluster-token.txt || true

    echo "Backed up: ${NODE_NAME}"
done

tar czf /backup/node-config-$(date +%Y%m%d).tar.gz \
    -C /backup node-config-$(date +%Y%m%d)/
```

## Step 4: Upload Backups to External Storage

```bash
#!/bin/bash
# upload-backups.sh - Upload configuration backups to S3

BACKUP_DATE=$(date +%Y%m%d)
S3_BUCKET="s3://my-harvester-config-backups"
AWS_REGION="us-east-1"

# Upload Kubernetes resource backup
aws s3 cp /backup/harvester-config-${BACKUP_DATE}.tar.gz \
    ${S3_BUCKET}/config/${BACKUP_DATE}/harvester-config.tar.gz

# Upload node configuration backup
aws s3 cp /backup/node-config-${BACKUP_DATE}.tar.gz \
    ${S3_BUCKET}/node-config/${BACKUP_DATE}/node-config.tar.gz

# Upload etcd snapshots (they're already in S3 if configured above)
# Or sync local snapshots:
aws s3 sync \
    /var/lib/rancher/rke2/server/db/snapshots/ \
    ${S3_BUCKET}/etcd-snapshots/ \
    --region ${AWS_REGION}

# List recent backups
aws s3 ls ${S3_BUCKET}/config/ --recursive | tail -20

echo "Backup upload complete"
```

## Step 5: Restore Configuration from Backup

To restore from an etcd snapshot after catastrophic failure:

```bash
# On a fresh Harvester installation (single node to start)
# Restore the etcd snapshot

# 1. Stop RKE2
sudo systemctl stop rke2-server

# 2. Copy the etcd snapshot to the node
scp backup-server:/backup/etcd-snapshot.db \
    /var/lib/rancher/rke2/server/db/snapshots/

# 3. Restore from the snapshot
sudo rke2 server --cluster-reset \
    --cluster-reset-restore-path=/var/lib/rancher/rke2/server/db/snapshots/manual-backup-20240104120000

# 4. Start RKE2
sudo systemctl start rke2-server

# 5. Verify the restore
kubectl get nodes
kubectl get vmi -A
```

## Step 6: Verify Backup Integrity

```bash
#!/bin/bash
# verify-backup.sh - Verify backup integrity

BACKUP_FILE="$1"
EXTRACT_DIR="/tmp/backup-verify-$(date +%s)"

mkdir -p ${EXTRACT_DIR}

echo "Verifying backup: ${BACKUP_FILE}"

# Extract and check content
tar xzf ${BACKUP_FILE} -C ${EXTRACT_DIR}

# Check key files exist
KEY_FILES=(
    "harvester-settings.yaml"
    "vm-images.yaml"
    "virtual-machines.yaml"
    "cluster-networks.yaml"
)

for FILE in "${KEY_FILES[@]}"; do
    if ls ${EXTRACT_DIR}/*/${FILE} 2>/dev/null | head -1 > /dev/null; then
        echo "[OK] ${FILE} found"
    else
        echo "[MISSING] ${FILE} not found in backup!"
    fi
done

# Count resources in backup
echo ""
echo "Resource counts:"
grep "^kind: " ${EXTRACT_DIR}/*/*.yaml | \
    awk '{print $2}' | sort | uniq -c | sort -rn

# Cleanup
rm -rf ${EXTRACT_DIR}
```

## Conclusion

A complete Harvester configuration backup strategy combines etcd snapshots (for rapid cluster recovery), exported Kubernetes resource definitions (for documentation and partial recovery), and node-level configuration files (for OS-level recovery). Store all backups in an external location — preferably S3 or another cloud storage — separate from the Harvester cluster itself. Test your restore procedures regularly; a backup that has never been tested is of uncertain value. With comprehensive configuration backups in place, you can confidently recover from hardware failures, misconfiguration events, or complete cluster loss.
