# How to Restore K3s from a Backup

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: K3s, Kubernetes, Backup, Disaster Recovery, DevOps

Description: Step-by-step instructions for restoring a K3s cluster from an etcd snapshot or SQLite backup after data loss or corruption.

## Introduction

When disaster strikes — whether from accidental deletion, data corruption, or a failed upgrade — having a reliable restore procedure is essential. K3s provides built-in tools to restore from etcd snapshots. This guide covers restoring from both SQLite and etcd snapshot backups, including single-node and HA cluster scenarios.

## Prerequisites

- A valid backup snapshot or SQLite database backup
- Root/sudo access to all K3s nodes
- `kubectl` access (or ability to re-establish it post-restore)
- All K3s services stopped during the restore process

## Scenario 1: Restore SQLite Database (Single-Node)

For single-server K3s deployments using the default SQLite backend:

```bash
# Step 1: Stop K3s
systemctl stop k3s

# Step 2: Verify the backup file exists
ls -lh /backup/k3s/20240315-120000/state.db

# Step 3: Move the corrupted database aside
mv /var/lib/rancher/k3s/server/db/state.db \
   /var/lib/rancher/k3s/server/db/state.db.corrupted

# Step 4: Copy the backup database into place
cp /backup/k3s/20240315-120000/state.db \
   /var/lib/rancher/k3s/server/db/state.db

# Step 5: Set proper permissions
chown root:root /var/lib/rancher/k3s/server/db/state.db
chmod 600 /var/lib/rancher/k3s/server/db/state.db

# Step 6: Start K3s
systemctl start k3s

# Step 7: Verify restoration
kubectl get nodes
kubectl get pods -A
```

## Scenario 2: Restore Embedded Etcd Snapshot (Single Server)

K3s has a built-in `etcd-snapshot restore` command:

```bash
# Step 1: Stop K3s on the server node
systemctl stop k3s

# Step 2: List available snapshots to find the one to restore
k3s etcd-snapshot list

# Step 3: Restore from a local snapshot
# The --cluster-reset flag resets etcd to a single-member cluster
k3s server \
  --cluster-reset \
  --cluster-reset-restore-path=/var/lib/rancher/k3s/server/db/snapshots/my-cluster-backup

# Wait for the reset to complete (process will exit when done)
```

After the reset completes, restart K3s normally:

```bash
# Step 4: Start K3s in normal mode
systemctl start k3s

# Step 5: Verify restoration
kubectl get nodes
kubectl get pods -A
```

## Scenario 3: Restore from S3 Snapshot

If your snapshots are stored in S3:

```bash
# Step 1: Stop K3s
systemctl stop k3s

# Step 2: List snapshots in S3
k3s etcd-snapshot list \
  --s3 \
  --s3-bucket=my-k3s-backups \
  --s3-region=us-east-1 \
  --s3-access-key=AKIAIOSFODNN7EXAMPLE \
  --s3-secret-key=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY

# Step 3: Restore from S3
k3s server \
  --cluster-reset \
  --cluster-reset-restore-path=<snapshot-name> \
  --etcd-s3 \
  --etcd-s3-bucket=my-k3s-backups \
  --etcd-s3-region=us-east-1 \
  --etcd-s3-access-key=AKIAIOSFODNN7EXAMPLE \
  --etcd-s3-secret-key=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY

# Step 4: Start K3s after reset completes
systemctl start k3s
```

## Scenario 4: Restore HA Cluster (Multiple Server Nodes)

For an HA cluster with multiple server nodes, the restore process requires coordinating all servers:

```bash
# On ALL server nodes, stop K3s
# Run this on server1, server2, server3...
systemctl stop k3s

# On the FIRST server node only, perform the restore
k3s server \
  --cluster-reset \
  --cluster-reset-restore-path=/var/lib/rancher/k3s/server/db/snapshots/my-cluster-backup

# Wait for completion, then start K3s on the first server
systemctl start k3s

# Verify server1 is healthy
kubectl get nodes

# On OTHER server nodes, delete the old etcd data directory
# and re-join the cluster
rm -rf /var/lib/rancher/k3s/server/db/etcd

# Restart K3s on other servers
# They will re-join and sync from the restored leader
systemctl start k3s
```

## Step 5: Restore Configuration Files

If certificates or tokens were also lost, restore them from backup:

```bash
# Restore TLS certificates
cp -r /backup/k3s/config-20240315/tls/* \
       /var/lib/rancher/k3s/server/tls/

# Restore node token
cp /backup/k3s/config-20240315/node-token \
   /var/lib/rancher/k3s/server/node-token

# Restore K3s configuration
cp /backup/k3s/config-20240315/config.yaml \
   /etc/rancher/k3s/config.yaml

# Fix permissions
chmod 600 /var/lib/rancher/k3s/server/node-token
```

## Step 6: Reconnect Agent Nodes

After restoring the server, agent nodes may need to be restarted to reconnect:

```bash
# On each agent node
systemctl restart k3s-agent

# Verify all agents have re-joined
kubectl get nodes
```

## Validation Checklist

After restoration, verify cluster health:

```bash
# Check all nodes are Ready
kubectl get nodes

# Check system pods
kubectl get pods -n kube-system

# Verify key resources are restored
kubectl get deployments -A
kubectl get services -A
kubectl get persistentvolumes

# Run a quick smoke test
kubectl run restore-test --image=nginx --restart=Never
kubectl wait --for=condition=Ready pod/restore-test --timeout=60s
kubectl delete pod restore-test
```

## Conclusion

K3s makes cluster restoration straightforward with its built-in `etcd-snapshot` tooling. The key to successful recovery is having recent, tested backups and a documented restore procedure. Always perform test restores in a non-production environment periodically to validate your backup strategy works when you need it most.
