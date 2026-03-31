# How to Achieve Crash-Consistent Replication with RBD Mirroring

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RBD, Mirroring, Consistency, Disaster Recovery

Description: Learn how to configure crash-consistent replication with RBD mirroring in Ceph, ensuring that replicated images can be recovered to a consistent state after a failure.

---

## Overview

Crash consistency means that even if the primary cluster fails mid-write, the replicated image on the secondary cluster represents a valid, consistent state - similar to what you would recover from after a hard reboot. Ceph achieves this through journal-based mirroring, which records writes in order before applying them to the image.

## Journal-Based Mirroring vs. Snapshot-Based Mirroring

Ceph supports two replication modes:

- **Journal-based** - continuous, write-ahead journal ensures crash consistency and minimal RPO
- **Snapshot-based** - periodic snapshots; crash consistency is limited to snapshot intervals

For true crash consistency, journal-based mirroring is required.

## Setting Up Journal-Based Mirroring

Enable journaling on the pool and images:

```bash
# Enable pool mirroring in pool mode (all images mirrored with journaling)
rbd mirror pool enable replicapool pool

# Verify pool mode
rbd mirror pool info replicapool
# Mode: pool
```

Create an image with journaling enabled (required for pool mode):

```bash
rbd create --size 20G \
  --image-feature layering,exclusive-lock,journaling \
  replicapool/critical-db

# Verify journaling is active
rbd info replicapool/critical-db | grep features
```

## Configuring Journal Commit Age

The journal commit age controls how frequently journal entries are flushed to disk:

```bash
# Reduce commit age for lower RPO (more frequent flushes)
rbd config image set replicapool/critical-db rbd_journal_commit_age 1

# View current setting
rbd config image get replicapool/critical-db rbd_journal_commit_age
```

Lower values reduce RPO but increase write latency.

## Configuring Journal Object Size

Large write workloads benefit from larger journal objects:

```bash
# Set journal order (object size = 2^order bytes, 24 = 16MB)
rbd config image set replicapool/critical-db rbd_journal_order 24
```

## Verifying Crash Consistency

Monitor the replication lag to understand current RPO:

```bash
# Check entries behind master
rbd mirror image status replicapool/critical-db --format json | \
  jq '.description'

# A low entries_behind_master value means minimal RPO
```

## Rook StorageClass for Crash-Consistent Replication

Configure the StorageClass to ensure all provisioned PVCs use journaling:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-ceph-block-crash-consistent
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
  clusterID: rook-ceph
  pool: replicapool
  imageFormat: "2"
  imageFeatures: layering,exclusive-lock,object-map,fast-diff,journaling
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
reclaimPolicy: Retain
```

## Testing Crash Consistency

Validate that the secondary cluster can recover a consistent image:

```bash
# On the secondary, after forced promotion
rbd mirror image promote --force replicapool/critical-db

# Map the image and run filesystem checks
rbd map replicapool/critical-db
fsck -n /dev/rbd0
```

## Summary

Crash-consistent replication with RBD mirroring requires journal-based mode, which ensures that all writes are recorded in order before application. Configure journaling as a default image feature in your StorageClass and tune `rbd_journal_commit_age` to balance RPO against write latency. Regularly monitor `entries_behind_master` to verify that replication lag stays within acceptable bounds.
