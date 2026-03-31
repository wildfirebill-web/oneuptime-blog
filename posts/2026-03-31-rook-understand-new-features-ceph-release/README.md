# How to Understand New Features in Each Ceph Release

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Release, Feature, Storage

Description: A practical guide to tracking and evaluating new features in each Ceph release, so you can make informed upgrade decisions for Rook-managed clusters.

---

## Overview

Ceph follows a major release cadence with named versions - Quincy, Reef, Squid, Tentacle. Each release brings new features, performance improvements, and changes that affect Rook deployments. Knowing how to evaluate these is essential for planning upgrades.

## Where to Find Release Notes

The primary sources for Ceph release information are:

```bash
# View release notes for currently running version
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph version
# Then check: https://docs.ceph.com/en/latest/releases/
```

Official channels to track:
- Ceph release notes at `docs.ceph.com/en/latest/releases/`
- Rook release notes at `rook.io/docs/rook/latest/`
- Ceph blog at `ceph.io/en/news/`

## Evaluating Features by Component

### RADOS / Core Storage

New RADOS features often affect data durability and performance:

```bash
# Check current RADOS features enabled in pools
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd pool get mypool all | grep pg_num
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph features
```

### RBD (Block Storage)

Evaluate new RBD features that may enhance snapshot and clone performance:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- rbd feature list
```

Enable a new RBD feature on existing images:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- rbd feature enable mypool/myimage deep-flatten
```

### CephFS

Track new MDS features with each release:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph fs dump
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph mds stat
```

### RGW (Object Storage)

New S3 API compatibility features are listed per release:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- radosgw-admin feature list
```

## Rook-Specific Feature Gates

Rook exposes new features through the CephCluster spec and feature flags:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
spec:
  cephVersion:
    image: quay.io/ceph/ceph:v19.2.0
  features:
    pg_autoscaler: true
```

Check which features Rook has enabled:

```bash
kubectl -n rook-ceph get cephcluster rook-ceph -o jsonpath='{.spec.features}'
```

## Tracking the Changelog Programmatically

Script to compare features between two releases:

```bash
#!/bin/bash
PREV_VER="18.2.0"
NEW_VER="19.2.0"
echo "Comparing Ceph $PREV_VER vs $NEW_VER"
curl -s "https://raw.githubusercontent.com/ceph/ceph/v${NEW_VER}/doc/releases/${NEW_VER%%.*}.rst" | head -100
```

## Summary

Tracking new Ceph features requires consulting official release notes, the Rook changelog, and using ceph CLI commands to inspect enabled features. By systematically reviewing each component - RADOS, RBD, CephFS, and RGW - you can identify high-value features to enable and plan informed upgrades for your Rook-managed cluster.
