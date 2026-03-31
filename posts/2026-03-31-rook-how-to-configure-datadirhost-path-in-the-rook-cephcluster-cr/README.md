# How to Configure dataDirHostPath in the Rook CephCluster CRD

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, CephCluster, Configuration, Kubernetes Storage

Description: Learn what dataDirHostPath does in the Rook CephCluster CRD, why it matters, and how to choose the right path for your storage nodes.

---

## What Is dataDirHostPath?

`dataDirHostPath` is a required field in the Rook `CephCluster` CRD that specifies the path on each Kubernetes node where Rook stores Ceph daemon configuration files, keyring files, and other runtime data.

This is a host path mount, meaning Rook mounts this directory from the node's filesystem into the Ceph daemon containers. It persists across pod restarts and ensures Ceph daemons can recover their state.

## Why dataDirHostPath Matters

Ceph daemons (MON, MGR, OSD) write configuration and key material to this directory. If the data in `dataDirHostPath` is lost (e.g., due to node reimaging or path change), Ceph daemons will not be able to recover their identity, leading to cluster health issues.

Key facts:
- MON daemons store their database under `{dataDirHostPath}/mon-{id}/data`
- Keyrings and config files for all daemons are written here
- The path must exist on every node where Ceph daemons run
- The directory must be writable by the Rook operator

## Basic Configuration Example

```yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  cephVersion:
    image: quay.io/ceph/ceph:v18.2.0
  dataDirHostPath: /var/lib/rook
  mon:
    count: 3
    allowMultiplePerNode: false
  storage:
    useAllNodes: true
    useAllDevices: true
```

The default value `/var/lib/rook` is appropriate for most deployments.

## Choosing the Right Path

When choosing `dataDirHostPath`, consider:

- **Disk space:** MON data can grow significantly. Ensure the filesystem hosting this path has enough free space.
- **Node reliability:** The path must survive reboots. Avoid ephemeral storage like `/tmp`.
- **Permissions:** Rook runs with elevated privileges and will create the directory if it does not exist.

For nodes with limited root disk space, use a dedicated disk or partition:

```bash
# On each storage node: create and prepare the data directory
mkdir -p /mnt/ceph-data/rook
chmod 755 /mnt/ceph-data/rook
```

Then configure:

```yaml
spec:
  dataDirHostPath: /mnt/ceph-data/rook
```

## Checking What Rook Stores There

After deployment, you can inspect the contents on any storage node:

```bash
ls -la /var/lib/rook/
```

Expected directories:

```text
rook-ceph/
  mon-a/
    data/
  mon-b/
    data/
  mon-c/
    data/
  osd-0/
  osd-1/
```

## Changing dataDirHostPath on a Live Cluster

Changing `dataDirHostPath` on a running cluster is dangerous and not supported as a live migration. The process requires:

1. Taking a cluster backup.
2. Scaling down all Ceph daemons.
3. Moving the data directory contents to the new path on all nodes.
4. Updating the CRD with the new path.
5. Restarting all daemons.

This is a high-risk operation. Plan your `dataDirHostPath` carefully before initial deployment.

## Disk Space Monitoring for dataDirHostPath

Monitor the filesystem hosting `dataDirHostPath` to avoid running out of space:

```bash
# Check disk usage of Rook data directory
du -sh /var/lib/rook/

# Check filesystem free space
df -h /var/lib/rook
```

Set up node alerts in Prometheus to warn when the filesystem exceeds 70-80% capacity.

## Impact on PVC-Based Deployments

For PVC-based cluster deployments (using `storageClassDeviceSets`), `dataDirHostPath` is still required for storing configuration and keyring data, even though the OSD data itself lives on PVCs. Both must be configured correctly.

## Summary

`dataDirHostPath` is a critical field in the Rook CephCluster CRD that defines where Ceph daemon configuration and key material are stored on each node. Choose a path on stable, persistent storage with sufficient disk space, and monitor the filesystem to prevent capacity issues. Changing this path post-deployment requires manual migration of data and a planned maintenance window, so select it carefully before deploying your cluster.
