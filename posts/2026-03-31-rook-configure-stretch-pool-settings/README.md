# How to Configure Stretch Pool Settings in Rook-Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Stretch Cluster, Pool, CRUSH Rule

Description: Learn how to configure Ceph pool settings for Rook stretch clusters, including CRUSH rules, replication across zones, and stretch-aware pool creation.

---

## Overview

In a Rook-Ceph stretch cluster, pools must be explicitly configured to distribute data across both physical sites. A standard replicated pool without a stretch CRUSH rule will not guarantee cross-site replication. This guide covers how to configure stretch pool settings using CephBlockPool CRDs and the Ceph toolbox.

## Understanding Stretch Pool Requirements

Stretch clusters use a modified CRUSH rule called `stretch_rule`. This rule ensures each object has copies in both zones plus satisfies a minimum replica requirement. For a 2+1 stretch setup, Ceph writes 4 copies total - 2 per site - but this can be configured.

The key pool parameters for stretch:

- `size`: total replicas (typically 4 for a 2+2 cross-site setup)
- `min_size`: minimum replicas needed to accept writes (typically 2)
- `crush_rule`: must reference the stretch CRUSH rule

## Creating a Stretch-Aware CephBlockPool

```yaml
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: stretch-pool
  namespace: rook-ceph
spec:
  replicated:
    size: 4
    requireSafeReplicaSize: true
  parameters:
    crush_rule: stretch_rule
    min_size: "2"
```

Apply it:

```bash
kubectl apply -f stretch-pool.yaml
```

## Verifying the CRUSH Rule

After creating the pool, confirm the correct CRUSH rule is applied:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool get stretch-pool crush_rule

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool get stretch-pool all
```

You should see `crush_rule: stretch_rule` in the output. Also verify the replication factor:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool get stretch-pool size
```

## Updating an Existing Pool for Stretch

If you have existing pools that were created before enabling stretch mode, you can update them:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool set existing-pool crush_rule stretch_rule

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool set existing-pool size 4

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool set existing-pool min_size 2
```

After updating, placement groups will start remapping to satisfy the new CRUSH rule. Monitor recovery progress:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph -w
```

## Configuring the StorageClass

Once the stretch pool is ready, create a StorageClass that references it:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-ceph-stretch-block
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
  clusterID: rook-ceph
  pool: stretch-pool
  imageFormat: "2"
  imageFeatures: layering
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
reclaimPolicy: Delete
allowVolumeExpansion: true
```

## Monitoring Pool Health

Verify placement group distribution across zones:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool stats stretch-pool

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph pg ls-by-pool stretch-pool | head -20
```

## Summary

Configuring stretch pool settings in Rook-Ceph requires setting the `stretch_rule` CRUSH rule and appropriate size/min_size parameters. Pools that use the default CRUSH rule will not replicate across both sites, leaving you vulnerable to site-level data loss. Always verify CRUSH rule assignment after pool creation and monitor placement group distribution to confirm cross-site replication is active.
