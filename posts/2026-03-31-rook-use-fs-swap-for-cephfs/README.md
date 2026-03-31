# How to Use fs swap for CephFS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, CephFS

Description: Learn how to use ceph fs swap to atomically exchange two CephFS filesystems, enabling zero-downtime migrations and blue-green filesystem deployments.

---

## What Is fs swap

`ceph fs swap` is a Ceph command that atomically exchanges the names of two CephFS filesystems. This allows you to perform a blue-green style deployment where you prepare a new filesystem with all the correct data and configuration, then swap it into production with a single atomic operation.

This command is available in Ceph Quincy (17.x) and later.

## Use Cases for fs swap

- **Blue-green filesystem deployment** - prepare a new filesystem offline, then swap it to production
- **Rolling back a bad migration** - quickly revert by swapping back to the previous filesystem
- **A/B testing** - alternate between two filesystem configurations without downtime
- **Disaster recovery testing** - swap a restored backup filesystem into production for validation

## Syntax

```bash
ceph fs swap <fs1> <fs2> --yes-i-really-mean-it
```

After the command, `fs1` takes the name of `fs2` and vice versa.

## Example: Blue-Green Filesystem Migration

Start with a production filesystem named `myfs` and a new filesystem named `myfs-new`:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash
```

```bash
# Verify both filesystems exist
ceph fs ls
```

Output:

```text
name: myfs, metadata pool: myfs-metadata, data pools: [myfs-data ]
name: myfs-new, metadata pool: myfs-new-metadata, data pools: [myfs-new-data ]
```

Prepare the new filesystem with all desired data and configuration, then swap:

```bash
ceph fs swap myfs myfs-new --yes-i-really-mean-it
```

After the swap:

```bash
ceph fs ls
```

Output:

```text
name: myfs, metadata pool: myfs-new-metadata, data pools: [myfs-new-data ]
name: myfs-new, metadata pool: myfs-metadata, data pools: [myfs-data ]
```

The name `myfs` now points to the pools that were `myfs-new`, and vice versa. Clients that mount by name (`-o fs=myfs`) automatically get the new filesystem.

## Swap with No Client Disruption

Because the swap is atomic, clients that are already mounted do not see the operation as a reconnect. New clients connecting to `myfs` will be routed to the new backing pools.

## Rolling Back

If the swap reveals a problem, roll back by swapping again:

```bash
ceph fs swap myfs myfs-new --yes-i-really-mean-it
```

This reverts the filesystem name mapping back to the original state.

## Rook Considerations

In Rook, `fs swap` operates at the Ceph layer and is not natively represented in the `CephFilesystem` CRD. After performing a swap:

1. The `CephFilesystem` CRD named `myfs` will now point to different pools than Rook expects
2. Restart the Rook operator to force re-reconciliation:

```bash
kubectl -n rook-ceph rollout restart deploy/rook-ceph-operator
```

3. Consider updating the `CephFilesystem` CRD to reflect the new pool assignments, or manage the swap entirely outside Rook for staging environments.

## Verifying the Swap

Confirm MDS daemons are active on the swapped filesystems:

```bash
ceph fs status myfs
ceph mds stat
```

## Summary

`ceph fs swap` atomically exchanges the names of two CephFS filesystems, enabling zero-downtime migrations and easy rollbacks. This is ideal for blue-green deployments where you prepare a new filesystem offline and then swap it into production. In Rook environments, manage the swap at the Ceph layer and restart the operator after swapping to ensure proper reconciliation.
