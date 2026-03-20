# How to Restore etcd Snapshots in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Restore, etcd

Description: Learn how to restore Rancher-managed clusters from etcd snapshots to recover from data loss or cluster corruption.

When a Kubernetes cluster experiences data corruption, accidental resource deletion, or a failed upgrade, restoring from an etcd snapshot can bring it back to a known good state. Rancher provides multiple methods to restore etcd snapshots for RKE and RKE2 clusters. This guide covers the complete restore process.

## Prerequisites

- A valid etcd snapshot (local or in S3)
- Admin access to Rancher
- SSH access to control plane nodes (for manual restoration)
- kubectl access to the management cluster

## Step 1: Identify Available Snapshots

### Via the Rancher UI

1. Navigate to the downstream cluster in Rancher.
2. Go to **Snapshots** in the cluster menu.
3. Review available snapshots with their timestamps.
4. Select the snapshot you want to restore from.

### Via kubectl

For RKE2 clusters:

```bash
kubectl get etcdsnapshots.rke.cattle.io -n fleet-default
```

For RKE clusters:

```bash
kubectl get etcdbackups.management.cattle.io
```

### On the Control Plane Node

```bash
# RKE2

ls -la /var/lib/rancher/rke2/server/db/snapshots/

# RKE
ls -la /opt/rke/etcd-snapshots/
```

## Step 2: Restore via the Rancher UI

The simplest way to restore is through the Rancher UI:

1. Navigate to the downstream cluster.
2. Go to **Snapshots**.
3. Find the snapshot you want to restore from.
4. Click the three-dot menu next to the snapshot.
5. Select **Restore**.
6. Confirm the restore operation.
7. Wait for the cluster to reconcile.

Rancher will handle stopping etcd, restoring the snapshot, and restarting the cluster components.

## Step 3: Restore RKE2 Clusters via CLI

If the Rancher UI is not available, you can restore directly on the control plane node.

### On a Single-Node RKE2 Cluster

SSH into the control plane node and stop RKE2:

```bash
systemctl stop rke2-server
```

Restore from a local snapshot:

```bash
rke2 server \
  --cluster-reset \
  --cluster-reset-restore-path=/var/lib/rancher/rke2/server/db/snapshots/SNAPSHOT_NAME
```

Start RKE2 again:

```bash
systemctl start rke2-server
```

### On a Multi-Node RKE2 Cluster

For high-availability clusters with multiple control plane nodes:

1. Stop RKE2 on all control plane nodes:

```bash
# Run on ALL control plane nodes
systemctl stop rke2-server
```

2. On the first control plane node, restore the snapshot:

```bash
rke2 server \
  --cluster-reset \
  --cluster-reset-restore-path=/var/lib/rancher/rke2/server/db/snapshots/SNAPSHOT_NAME
```

3. Remove the old etcd data on the other control plane nodes:

```bash
# Run on secondary control plane nodes
rm -rf /var/lib/rancher/rke2/server/db/etcd
```

4. Start RKE2 on the first node:

```bash
systemctl start rke2-server
```

5. Once the first node is healthy, start the other nodes:

```bash
systemctl start rke2-server
```

## Step 4: Restore from an S3 Snapshot

If your snapshots are stored in S3:

```bash
rke2 server \
  --cluster-reset \
  --etcd-s3 \
  --etcd-s3-bucket=etcd-snapshots \
  --etcd-s3-access-key=YOUR_ACCESS_KEY \
  --etcd-s3-secret-key=YOUR_SECRET_KEY \
  --etcd-s3-region=us-east-1 \
  --etcd-s3-folder=cluster-1 \
  --cluster-reset-restore-path=SNAPSHOT_NAME
```

## Step 5: Restore RKE Clusters via Rancher API

For RKE clusters, you can trigger a restore via the Rancher API:

```bash
curl -X POST \
  'https://rancher.yourdomain.com/v3/clusters/CLUSTER_ID?action=restoreFromEtcdBackup' \
  -H 'Authorization: Bearer YOUR_API_TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{
    "etcdBackupId": "BACKUP_ID"
  }'
```

Get the backup ID from:

```bash
curl -s \
  'https://rancher.yourdomain.com/v3/etcdbackups?clusterId=CLUSTER_ID' \
  -H 'Authorization: Bearer YOUR_API_TOKEN' | jq '.data[].id'
```

## Step 6: Verify the Restore

After the restore completes, verify the cluster is healthy:

```bash
kubectl get nodes
kubectl get pods -A
kubectl get cs
```

Check etcd health:

```bash
kubectl -n kube-system exec -it etcd-NODE_NAME -- \
  etcdctl endpoint health \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

Verify workloads are running as expected:

```bash
kubectl get deployments -A
kubectl get statefulsets -A
```

## Step 7: Post-Restore Tasks

After a successful restore:

1. **Verify application connectivity**: Test that applications are serving traffic correctly.

2. **Check persistent volumes**: Ensure PVs are bound and accessible:

```bash
kubectl get pv
kubectl get pvc -A
```

3. **Reconcile any changes made after the snapshot**: Any changes made between the snapshot time and the restore time will be lost. Review and reapply if needed.

4. **Take a new snapshot**: Create a fresh snapshot to mark the new baseline:

```bash
rke2 etcd-snapshot save --name post-restore-$(date +%Y%m%d%H%M)
```

5. **Verify Rancher connectivity**: Ensure the cluster reconnects to the Rancher management server:

```bash
kubectl get pods -n cattle-system
```

## Troubleshooting

### Cluster Stuck in Updating State

After a restore, the cluster may appear stuck in Rancher. Wait 10-15 minutes for reconciliation. If it persists, restart the cattle-cluster-agent:

```bash
kubectl rollout restart deployment cattle-cluster-agent -n cattle-system
```

### etcd Quorum Lost

On multi-node clusters, if quorum is lost, you must restore on a single node first and then re-join the others. Follow the multi-node restore procedure in Step 3.

### Snapshot Corrupted

If a snapshot fails to restore, try an older snapshot. Verify snapshot integrity before restoring:

```bash
ETCDCTL_API=3 etcdctl snapshot status SNAPSHOT_FILE --write-out=table
```

### Nodes Not Rejoining

If worker nodes do not rejoin after the restore, restart the kubelet on each worker:

```bash
systemctl restart rke2-agent  # or kubelet for RKE
```

## Conclusion

Restoring etcd snapshots is a critical operation that can save your cluster from data loss. Whether you use the Rancher UI, CLI, or API, the process involves stopping the cluster, applying the snapshot, and verifying the restored state. Practice this procedure in a non-production environment so your team is ready when a real recovery is needed.
