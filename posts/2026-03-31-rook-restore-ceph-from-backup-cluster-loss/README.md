# How to Restore Ceph from Backup After Cluster Loss

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Disaster Recovery, Backup, Restore, Storage

Description: Learn how to restore a Ceph cluster from backup after complete cluster loss, covering monitor database restoration, OSD data recovery, and Rook-managed cluster rebuilding.

---

## Understanding Ceph Backup Components

A complete Ceph backup includes:
1. Monitor database (cluster state and configuration)
2. OSD data (actual stored data)
3. Ceph keyrings (authentication)
4. CephFS journal/metadata pool (if CephFS is used)

For Rook-managed clusters, Kubernetes secrets and CRD specifications are also essential.

## Step 1: Restore the New Cluster Infrastructure

Set up fresh nodes with the same or equivalent hardware. Install required packages:

```bash
apt-get install -y ceph ceph-osd ceph-mon ceph-mgr
```

For Rook, ensure the Kubernetes cluster is running and install Rook:

```bash
kubectl create -f crds.yaml -f common.yaml -f operator.yaml
```

## Step 2: Restore Monitor Database

Copy the monitor backup to the first monitor node:

```bash
# Extract monitor backup
tar -xzf /backup/mon-a-20260310.tar.gz -C /

# Fix ownership
chown -R ceph:ceph /var/lib/ceph/mon/ceph-a/

# Start the monitor
systemctl start ceph-mon@a
```

## Step 3: Restore Keyrings

Restore the cluster keyring to all nodes:

```bash
cp /backup/ceph.client.admin.keyring /etc/ceph/
cp /backup/ceph.keyring /etc/ceph/
chmod 600 /etc/ceph/*.keyring
```

## Step 4: Re-register OSDs

If OSD disks survived, recover them by re-activating existing OSDs:

```bash
# Scan for existing OSD data
ceph-volume lvm list

# Activate recovered OSD
ceph-volume lvm activate <osd-id> <osd-fsid>
```

For all OSDs on a node:

```bash
ceph-volume lvm activate --all
```

## Step 5: Verify Cluster Recovery

Check that monitors have quorum and OSDs are joining:

```bash
ceph mon stat
ceph osd stat
ceph -s
```

Wait for PGs to go `active+clean`:

```bash
watch -n 5 ceph -s
```

## Rook Cluster Restoration

For Rook-managed clusters, restore Kubernetes secrets first:

```bash
# Restore Rook/Ceph secrets from backup
kubectl apply -f /backup/rook-ceph-secrets.yaml

# Restore the CephCluster CRD spec
kubectl apply -f /backup/cephcluster.yaml
```

Let the Rook operator reconcile the cluster:

```bash
kubectl -n rook-ceph get pods -w
```

## Restoring RBD Images

If RBD images were backed up externally:

```bash
# Recreate pool
ceph osd pool create replicapool 32 32
ceph osd pool application enable replicapool rbd

# Import RBD image
rbd import /backup/myimage-20260310.raw replicapool/myimage
```

## Restoring from Object Store Backup (RGW)

For object store data backed up to S3:

```bash
# Create object store and sync data
rclone sync s3://backup-bucket/rgw-data \
  ceph-rgw:target-bucket \
  --s3-endpoint http://ceph-rgw:80
```

## Summary

Restoring a Ceph cluster from backup requires restoring the monitor database, keyrings, and OSD data in sequence. Physical OSD disks that survived cluster loss often retain their data and can be reactivated. In Rook environments, restoring Kubernetes secrets and CRD specs followed by operator reconciliation automates much of the recovery process. Regular backups of monitor databases, keyrings, and RBD images are essential for complete recoverability.
