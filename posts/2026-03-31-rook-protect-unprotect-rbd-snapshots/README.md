# How to Protect and Unprotect RBD Snapshots

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RBD, Snapshot, Data Protection, Clone

Description: Learn how to protect and unprotect RBD snapshots in Ceph, understand why protection is required before cloning, and manage snapshot lifecycle with protection policies.

---

## Why Protect Snapshots?

RBD snapshot protection is a mechanism that prevents accidental deletion of snapshots that have active clones depending on them. A snapshot must be protected before it can be used as a clone source. Attempting to delete a protected snapshot fails with an error, preventing orphaned clones.

## When Protection is Required

- Before creating a clone from a snapshot
- When a snapshot is used as a golden image template
- When Kubernetes VolumeSnapshot content references an RBD snapshot

## Protecting a Snapshot

```bash
# Create image and snapshot
rbd create mypool/golden-image --size 20G
rbd snap create mypool/golden-image@v1

# Protect the snapshot
rbd snap protect mypool/golden-image@v1
```

Verify protection status:

```bash
rbd snap ls mypool/golden-image
```

Output:
```text
SNAPID  NAME  SIZE    PROTECTED  TIMESTAMP
1       v1    20 GiB  yes        Tue Mar 31 10:00:00 2026
```

## Attempting to Delete a Protected Snapshot (Will Fail)

```bash
rbd snap rm mypool/golden-image@v1
```

Error:
```yaml
librbd: snapshot 'v1' is protected
rbd: failed to remove snapshot: (16) Device or resource busy
```

## Creating Clones from Protected Snapshots

```bash
rbd clone mypool/golden-image@v1 mypool/app1-clone
rbd clone mypool/golden-image@v1 mypool/app2-clone
```

Check clones:

```bash
rbd children mypool/golden-image@v1
```

## Unprotecting a Snapshot

You can only unprotect a snapshot after all clones derived from it are either flattened or deleted:

```bash
# Flatten clones first
rbd flatten mypool/app1-clone
rbd flatten mypool/app2-clone

# Then unprotect
rbd snap unprotect mypool/golden-image@v1
```

If clones still exist:

```bash
rbd snap unprotect mypool/golden-image@v1
```

Error:
```yaml
librbd: snapshot 'v1' has children
```

## Bulk Unprotect Workflow

Script to safely unprotect and delete a snapshot:

```bash
#!/bin/bash
POOL="mypool"
IMAGE="golden-image"
SNAP="v1"

# Flatten all children
for child in $(rbd children $POOL/$IMAGE@$SNAP); do
  echo "Flattening $child..."
  rbd flatten "$child"
done

# Unprotect
rbd snap unprotect $POOL/$IMAGE@$SNAP

# Delete
rbd snap rm $POOL/$IMAGE@$SNAP
echo "Snapshot $SNAP deleted"
```

## Checking All Protected Snapshots in a Pool

```bash
for image in $(rbd ls mypool); do
  rbd snap ls mypool/$image | grep " yes " | while read snap; do
    echo "$image: $snap"
  done
done
```

## Integration with Kubernetes VolumeSnapshot

When Rook CSI creates a clone-based PVC from a VolumeSnapshot, it automatically protects the underlying RBD snapshot. When the VolumeSnapshot is deleted, Rook unprotects and cleans up the snapshot.

## Summary

RBD snapshot protection prevents accidental deletion of snapshots that active clones depend on. Always protect a snapshot with `rbd snap protect` before creating clones. To delete a protected snapshot, first flatten all clones with `rbd flatten`, then run `rbd snap unprotect` followed by `rbd snap rm`. Rook CSI manages protection automatically for Kubernetes VolumeSnapshot objects.
