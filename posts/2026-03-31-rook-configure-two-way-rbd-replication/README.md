# How to Configure Two-Way RBD Replication

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RBD, Mirroring, Replication

Description: Learn how to configure two-way RBD mirroring in Rook-Ceph so both clusters can act as primary or secondary with active-active or active-passive replication.

---

## Two-Way RBD Replication Overview

Two-way RBD mirroring allows both clusters to replicate to each other. Unlike one-way mirroring, either cluster can serve as the primary for different images, enabling:

- Active-active workload distribution across two clusters
- Seamless failover and failback without re-seeding data
- Bidirectional disaster recovery

Two-way mirroring requires careful management to avoid split-brain scenarios where both clusters think they own the same image.

## Step 1 - Enable Mirroring on Both Clusters

Enable pool-level mirroring on both the primary and secondary clusters:

```bash
# On primary
kubectl exec -it deploy/rook-ceph-tools -n rook-ceph -- \
  rbd mirror pool enable replicapool pool

# On secondary
kubectl exec -it deploy/rook-ceph-tools -n rook-ceph-secondary -- \
  rbd mirror pool enable replicapool pool
```

## Step 2 - Generate Bootstrap Tokens from Both Sites

Generate a token from the primary:

```bash
kubectl exec -it deploy/rook-ceph-tools -n rook-ceph -- \
  rbd mirror pool peer bootstrap create \
  --site-name site-a replicapool > /tmp/site-a-token.txt
```

Generate a token from the secondary:

```bash
kubectl exec -it deploy/rook-ceph-tools -n rook-ceph-secondary -- \
  rbd mirror pool peer bootstrap create \
  --site-name site-b replicapool > /tmp/site-b-token.txt
```

## Step 3 - Import Tokens in Both Directions

Import site-b token into site-a (for bidirectional):

```bash
SITE_B_TOKEN=$(cat /tmp/site-b-token.txt)
kubectl exec -it deploy/rook-ceph-tools -n rook-ceph -- \
  rbd mirror pool peer bootstrap import \
  --site-name site-a \
  replicapool "${SITE_B_TOKEN}"
```

Import site-a token into site-b:

```bash
SITE_A_TOKEN=$(cat /tmp/site-a-token.txt)
kubectl exec -it deploy/rook-ceph-tools -n rook-ceph-secondary -- \
  rbd mirror pool peer bootstrap import \
  --site-name site-b \
  replicapool "${SITE_A_TOKEN}"
```

## Step 4 - Deploy rbd-mirror Daemons on Both Sites

Deploy the mirror daemon on both clusters:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephRBDMirror
metadata:
  name: rbd-mirror
  namespace: rook-ceph
spec:
  count: 1
  peers:
    secretNames:
    - rbd-peer-token
```

Apply on both clusters.

## Step 5 - Assign Images to Specific Primary Sites

In two-way mirroring, each image has a primary site. Assign images to site-a:

```bash
kubectl exec -it deploy/rook-ceph-tools -n rook-ceph -- \
  rbd mirror image enable replicapool/myimage snapshot

kubectl exec -it deploy/rook-ceph-tools -n rook-ceph -- \
  rbd mirror image status replicapool/myimage
```

## Step 6 - Verify Bidirectional Sync

Check mirror status from each site:

```bash
# From site-a
kubectl exec -it deploy/rook-ceph-tools -n rook-ceph -- \
  rbd mirror pool status replicapool --verbose

# From site-b
kubectl exec -it deploy/rook-ceph-tools -n rook-ceph-secondary -- \
  rbd mirror pool status replicapool --verbose
```

```text
health: OK
images: 10 total
    5 primary
    5 replaying
```

## Step 7 - Preventing Split-Brain

Enforce that each image has only one primary at a time. Use application-level coordination to ensure only the primary site's application is writing to a given image. Monitor with:

```bash
rbd mirror image status replicapool/myimage | grep primary
```

## Summary

Two-way RBD mirroring in Rook-Ceph enables bidirectional replication by importing bootstrap tokens in both directions and deploying `rbd-mirror` on both clusters. Each image has a designated primary site, with the other site holding a read-only replica. This setup supports both active-active workload distribution and seamless failover without re-seeding, but requires careful application-level coordination to prevent split-brain.
