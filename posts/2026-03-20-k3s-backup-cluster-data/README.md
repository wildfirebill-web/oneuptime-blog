# How to Back Up K3s Cluster Data

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: K3s, Kubernetes, Backup, Disaster Recovery, DevOps

Description: Learn how to back up K3s cluster data including etcd snapshots, SQLite databases, and critical configuration files.

## Introduction

Data loss in a Kubernetes cluster can be catastrophic — losing cluster state means losing all deployed workloads, configurations, and secrets. K3s supports two datastore backends: **SQLite** (default for single-node) and **embedded etcd** (for HA clusters). Both require different backup strategies. This guide covers backing up all critical K3s data.

## Understanding K3s Data Stores

K3s stores cluster state in one of two ways:

- **SQLite** (`/var/lib/rancher/k3s/server/db/state.db`): Default for single-server deployments
- **Embedded etcd** (`/var/lib/rancher/k3s/server/db/etcd/`): Used for HA multi-server deployments
- **External datastore**: PostgreSQL, MySQL, or etcd — managed externally

## Prerequisites

- Root or sudo access to the K3s server node
- Sufficient disk space for backups
- (Optional) Remote storage: S3, NFS, or SFTP

## Step 1: Back Up SQLite Database

For single-node K3s installations using the default SQLite datastore:

```bash
# Stop K3s to ensure data consistency (optional but recommended)
systemctl stop k3s

# Create backup directory with timestamp
BACKUP_DIR="/backup/k3s/$(date +%Y%m%d-%H%M%S)"
mkdir -p "$BACKUP_DIR"

# Copy the SQLite database
cp /var/lib/rancher/k3s/server/db/state.db "$BACKUP_DIR/state.db"

# Also back up the cred directory (tokens, certificates)
cp -r /var/lib/rancher/k3s/server/cred "$BACKUP_DIR/cred"

# Restart K3s
systemctl start k3s

echo "SQLite backup saved to: $BACKUP_DIR"
```

For a live backup without stopping K3s, use SQLite's `.backup` command:

```bash
# Live backup using sqlite3
sqlite3 /var/lib/rancher/k3s/server/db/state.db \
  ".backup /backup/k3s/state-$(date +%Y%m%d).db"
```

## Step 2: Back Up Embedded Etcd Using K3s Snapshots

K3s has built-in etcd snapshot functionality:

```bash
# Take an on-demand etcd snapshot
k3s etcd-snapshot save --name my-cluster-backup

# By default, snapshots are saved to:
# /var/lib/rancher/k3s/server/db/snapshots/

# List available snapshots
k3s etcd-snapshot list

# View snapshot details
ls -lh /var/lib/rancher/k3s/server/db/snapshots/
```

### Scheduled Snapshots

K3s can automatically take scheduled etcd snapshots. Configure via the K3s config file:

```yaml
# /etc/rancher/k3s/config.yaml
etcd-snapshot-schedule-cron: "0 */6 * * *"   # Every 6 hours
etcd-snapshot-retention: 10                   # Keep last 10 snapshots
etcd-snapshot-dir: /backup/k3s/etcd-snapshots # Custom snapshot directory
```

Or pass flags to the K3s service:

```bash
# /etc/systemd/system/k3s.service.env
K3S_ETCD_SNAPSHOT_SCHEDULE_CRON=0 */6 * * *
K3S_ETCD_SNAPSHOT_RETENTION=10
```

## Step 3: Back Up Critical Configuration Files

Several configuration files are essential for cluster recovery:

```bash
#!/bin/bash
# backup-k3s-config.sh

BACKUP_DIR="/backup/k3s/config-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$BACKUP_DIR"

# K3s server configuration
cp /etc/rancher/k3s/config.yaml "$BACKUP_DIR/" 2>/dev/null || true

# TLS certificates and keys
cp -r /var/lib/rancher/k3s/server/tls "$BACKUP_DIR/tls"

# Node token (needed to add new nodes)
cp /var/lib/rancher/k3s/server/node-token "$BACKUP_DIR/node-token"

# Static manifests (auto-deployed resources)
cp -r /var/lib/rancher/k3s/server/manifests "$BACKUP_DIR/manifests"

# Kubeconfig
cp /etc/rancher/k3s/k3s.yaml "$BACKUP_DIR/k3s.yaml"

echo "Configuration backup saved to: $BACKUP_DIR"
```

## Step 4: Upload Snapshots to S3

For off-node storage, K3s supports S3-compatible storage natively:

```bash
# /etc/rancher/k3s/config.yaml
etcd-s3: true
etcd-s3-bucket: my-k3s-backups
etcd-s3-region: us-east-1
etcd-s3-access-key: AKIAIOSFODNN7EXAMPLE
etcd-s3-secret-key: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
etcd-s3-endpoint: s3.amazonaws.com
```

Or trigger an S3 snapshot manually:

```bash
k3s etcd-snapshot save \
  --s3 \
  --s3-bucket=my-k3s-backups \
  --s3-region=us-east-1 \
  --s3-access-key=AKIAIOSFODNN7EXAMPLE \
  --s3-secret-key=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY \
  --name=manual-backup
```

## Step 5: Automate Backups with Cron

Create a comprehensive backup script and schedule it:

```bash
#!/bin/bash
# /usr/local/bin/k3s-backup.sh

set -euo pipefail

BACKUP_ROOT="/backup/k3s"
TIMESTAMP=$(date +%Y%m%d-%H%M%S)
BACKUP_DIR="$BACKUP_ROOT/$TIMESTAMP"

mkdir -p "$BACKUP_DIR"

# Take etcd snapshot
k3s etcd-snapshot save --name "scheduled-$TIMESTAMP"

# Copy snapshot to backup dir
cp /var/lib/rancher/k3s/server/db/snapshots/scheduled-"$TIMESTAMP"* "$BACKUP_DIR/"

# Back up config files
cp /etc/rancher/k3s/config.yaml "$BACKUP_DIR/" 2>/dev/null || true
cp /var/lib/rancher/k3s/server/node-token "$BACKUP_DIR/"

# Remove backups older than 7 days
find "$BACKUP_ROOT" -maxdepth 1 -type d -mtime +7 -exec rm -rf {} \;

echo "Backup completed: $BACKUP_DIR"
```

Add to crontab:

```bash
# Run backup every 6 hours
echo "0 */6 * * * root /usr/local/bin/k3s-backup.sh >> /var/log/k3s-backup.log 2>&1" \
  > /etc/cron.d/k3s-backup
```

## Conclusion

A robust backup strategy for K3s should include both the datastore (SQLite or etcd snapshots) and critical configuration files like certificates and node tokens. Use K3s's built-in S3 snapshot support for off-site backups, and always test your restore procedure periodically to ensure backups are valid and recovery is possible.
