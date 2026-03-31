# How to Upgrade Ceph Version Through Rook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Upgrade, Kubernetes, Cluster Management

Description: Learn how to upgrade the Ceph version in a Rook-managed cluster by updating the container image tag and monitoring the rolling upgrade process.

---

## Overview

Rook makes upgrading Ceph straightforward by handling the rolling upgrade of daemons automatically. When you update the Ceph container image in the CephCluster CRD, Rook orchestrates the upgrade in the correct order: monitors, managers, OSDs, MDS, and RGW daemons.

## Prerequisites

Before upgrading:
- The cluster must be in `HEALTH_OK` state
- All PGs must be in `active+clean` state
- Review the Ceph and Rook release notes for any special upgrade instructions

Check cluster health:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph status
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph health detail
```

## Step 1 - Identify the Target Version

Check available Ceph container images at `quay.io/ceph/ceph`. Choose a supported version based on your Rook version:

| Rook Version | Supported Ceph Versions |
|---|---|
| v1.14 | v18 (Reef), v17 (Quincy) |
| v1.15 | v19 (Squid), v18 (Reef) |

Check the current version:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph version
```

## Step 2 - Update the CephCluster Image

Edit the CephCluster to update the image tag:

```bash
kubectl -n rook-ceph edit cephcluster rook-ceph
```

Change the `cephVersion.image` field:

```yaml
spec:
  cephVersion:
    image: quay.io/ceph/ceph:v18.2.4
    allowUnsupported: false
```

Alternatively, patch it directly:

```bash
kubectl -n rook-ceph patch cephcluster rook-ceph \
  --type merge \
  -p '{"spec":{"cephVersion":{"image":"quay.io/ceph/ceph:v18.2.4"}}}'
```

## Step 3 - Monitor the Upgrade

Watch the pods restart with the new image:

```bash
kubectl -n rook-ceph get pods -w
```

Monitor the upgrade progress through the operator logs:

```bash
kubectl -n rook-ceph logs -f deploy/rook-ceph-operator | grep -i upgrade
```

The operator upgrades daemons in this order:
1. Monitors (one at a time)
2. Managers
3. OSDs (by default, one at a time)
4. MDS daemons
5. RGW daemons

## Step 4 - Verify Upgrade Progress

Check the Ceph version after each daemon type upgrades:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph versions
```

Expected output showing the new version:

```text
{
    "mon": {
        "ceph version 18.2.4 ...": 3
    },
    "mgr": {
        "ceph version 18.2.4 ...": 1
    },
    "osd": {
        "ceph version 18.2.4 ...": 6
    },
    "mds": {
        "ceph version 18.2.4 ...": 2
    }
}
```

## Step 5 - Verify Cluster Health Post-Upgrade

Confirm the cluster is healthy after the upgrade completes:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph status
kubectl -n rook-ceph get cephcluster rook-ceph -o jsonpath='{.status.phase}'
```

## Handling Upgrade Issues

If an OSD fails to upgrade, check the pod logs:

```bash
kubectl -n rook-ceph logs rook-ceph-osd-0-xxx | tail -20
```

If a daemon crashes with the new image, roll back by reverting the image tag:

```bash
kubectl -n rook-ceph patch cephcluster rook-ceph \
  --type merge \
  -p '{"spec":{"cephVersion":{"image":"quay.io/ceph/ceph:v18.2.2"}}}'
```

## Upgrading Rook Operator First

If upgrading both Rook and Ceph, always upgrade the Rook operator first:

```bash
kubectl -n rook-ceph set image deploy/rook-ceph-operator \
  rook-ceph-operator=rook/ceph:v1.15.0
```

Wait for the operator to be ready before updating the Ceph image.

## Summary

Upgrading Ceph through Rook requires updating the `cephVersion.image` field in the CephCluster CRD. Rook's operator then performs a rolling upgrade of all daemons in dependency order. Monitor progress through `kubectl logs` and `ceph versions` to verify each daemon type upgrades successfully. Always ensure the cluster is healthy before starting the upgrade and verify health again afterward.
