# How to Flatten RBD Clones to Remove Snapshot Dependencies

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RBD, Clone, Flatten, Snapshot, Performance

Description: Learn how to flatten RBD clones in Ceph to remove their dependency on parent snapshots, improve performance, and enable deletion of parent images and snapshots.

---

## Why Flatten RBD Clones?

RBD clones share data with their parent snapshot using copy-on-write. While this is efficient at creation time, deep clone chains cause performance issues:

- **Read amplification** - reads must traverse the clone chain to parent snapshots
- **Parent dependency** - parent snapshot cannot be deleted while clones exist
- **Chain depth limit** - Ceph recommends keeping clone chains below 5 levels

Flattening copies all parent data into the clone, making it fully independent.

## Understanding the Clone Chain

```text
golden-image@v1 (snapshot, protected)
  |-- clone-app1 (writes diverge here)
  |-- clone-app2
        |-- clone-app2-backup@snap1
              |-- nested-clone (deep chain - avoid this)
```

## Checking Clone Chain Depth

```bash
rbd info mypool/clone-app1 | grep "parent"
```

Output:
```yaml
parent: mypool/golden-image@v1
overlap: 20 GiB
```

## Flattening a Clone

```bash
rbd flatten mypool/clone-app1
```

Flattening copies ~20GB of data from parent to clone. Monitor progress:

```bash
rbd status mypool/clone-app1
```

Output during flatten:
```yaml
Watchers: none
Migration:
  source: mypool/golden-image@v1 (snap_id: 1)
  destination: mypool/clone-app1 (id: abc123)
  state: executing (12%)
```

## Running Flatten in the Background

For large images, run flatten asynchronously:

```bash
rbd flatten mypool/clone-app1 &
FLATTEN_PID=$!

# Monitor progress
while kill -0 $FLATTEN_PID 2>/dev/null; do
  rbd status mypool/clone-app1 | grep "executing"
  sleep 10
done
echo "Flatten complete"
```

## Flattening CSI-Managed Clones in Kubernetes

For Rook CSI, configure the StorageClass to automatically flatten clones upon restore:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-ceph-block
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
  clusterID: rook-ceph
  pool: mypool
  imageFeatures: layering
  csi.storage.k8s.io/controller-expand-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/controller-expand-secret-namespace: rook-ceph
  flattenImageEnabled: "true"
```

## Verifying a Clone is Flattened

After flattening, the clone should have no parent:

```bash
rbd info mypool/clone-app1 | grep "parent"
```

If there is no output, the clone is fully independent.

## Freeing Up the Parent Snapshot

Once all clones are flattened, you can delete the parent snapshot:

```bash
# Unprotect (required if protected)
rbd snap unprotect mypool/golden-image@v1

# Delete the snapshot
rbd snap rm mypool/golden-image@v1
```

If clones still depend on it, you'll get:
```yaml
librbd: snapshot 'v1' is protected
```

## Performance Impact of Flattening

- During flatten: high write IOPS to the OSD pool containing the clone
- After flatten: read performance improves because the clone reads locally
- Storage: clone now uses full capacity; no sharing benefit after flatten

## Summary

Flattening RBD clones copies all parent snapshot data into the clone image, making it independent and allowing parent snapshot deletion. Use `rbd flatten` for manual flattening, or enable `flattenImageEnabled: "true"` in the Rook StorageClass for automatic post-provision flattening. Monitor flatten progress with `rbd status` and verify completion with `rbd info`. Flatten deep clone chains proactively to maintain read performance.
