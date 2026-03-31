# How to Clean Up OSD Data on Disks Before Reuse in Rook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Kubernetes, OSD, Storage

Description: Safely wipe OSD data from disks before reusing them in a new Rook-Ceph cluster or repurposing them for another workload.

---

## Why Disk Cleanup Is Required

When a disk was previously used as a Ceph OSD, it contains BlueStore metadata, LVM labels, and partition signatures. If you attempt to reuse the disk without cleaning it, Rook will skip the device during OSD provisioning and log a warning about it already containing data.

You must wipe these signatures before the disk can be re-provisioned.

## Step 1: Remove the OSD from the Running Cluster

Before wiping a disk, the OSD must be removed gracefully from the cluster. Identify the OSD ID associated with the disk:

```bash
kubectl -n rook-ceph exec -it <toolbox-pod> -- \
  ceph osd tree
```

Mark the OSD out and wait for data to migrate away:

```bash
kubectl -n rook-ceph exec -it <toolbox-pod> -- \
  ceph osd out osd.<id>
```

Wait until the cluster is back to `HEALTH_OK` or all PGs are active+clean:

```bash
kubectl -n rook-ceph exec -it <toolbox-pod> -- \
  ceph -w
```

Then purge the OSD:

```bash
kubectl -n rook-ceph exec -it <toolbox-pod> -- \
  ceph osd purge osd.<id> --yes-i-really-mean-it
```

## Step 2: Use the Rook Cleanup Job

Rook provides a disk cleanup mechanism via the `cleanupPolicy` in the `CephCluster`. For targeted disk cleanup, you can run the cleanup script on the node directly.

Label the node and use a DaemonSet-style job (or manually SSH to the node):

```bash
# On the node that holds the disk
sudo sgdisk --zap-all /dev/sdb
sudo dd if=/dev/zero of=/dev/sdb bs=4096 count=100 oflag=direct,dsync
sudo blkdiscard /dev/sdb
```

## Step 3: Remove LVM and Device Mapper Artifacts

BlueStore OSDs may leave LVM volumes on the disk:

```bash
sudo lvs | grep ceph
sudo vgremove <ceph-vg-name>
sudo pvremove /dev/sdb
```

Remove any device mapper entries:

```bash
sudo dmsetup ls | grep ceph
sudo dmsetup remove <ceph-dm-entry>
```

## Step 4: Verify the Disk Is Clean

Check that no Ceph signatures remain:

```bash
sudo lsblk -f /dev/sdb
sudo file -s /dev/sdb
```

The output should show no filesystem or partition signatures. Also verify with `blkid`:

```bash
sudo blkid /dev/sdb
```

No output means the disk is clean.

## Step 5: Reprovisioning the Disk in Rook

Once the disk is clean, add it back to the `CephCluster` spec:

```yaml
spec:
  storage:
    nodes:
      - name: worker-node-1
        devices:
          - name: sdb
```

Apply the update and watch Rook provision a new OSD on the disk:

```bash
kubectl apply -f cluster.yaml
kubectl -n rook-ceph get pods -l app=rook-ceph-osd -w
```

## Automating Cleanup with Rook's Built-in Policy

When you delete the entire `CephCluster`, Rook can clean disks automatically if you set:

```yaml
spec:
  cleanupPolicy:
    confirmation: yes-really-destroy-data
    sanitizeDisks:
      method: complete
      dataSource: zero
      iteration: 1
```

This is destructive and intended for full cluster teardown only.

## Summary

Reusing OSD disks in Rook requires removing the OSD from the cluster, waiting for data migration, purging the OSD from Ceph, and then wiping LVM labels and partition signatures with `sgdisk`, `dd`, and `dmsetup`. Only after all Ceph metadata is removed will Rook detect the disk as available and provision a new OSD on it.
