# How to Understand Deprecated Features in Ceph Reef

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Deprecation, Reef, Storage

Description: Learn which features are deprecated in Ceph Reef (18.x), why they are being removed, and how to migrate before upgrading to future releases.

---

## Overview

Ceph Reef (18.x) marked several features as deprecated in preparation for Squid and future releases. Understanding these deprecations helps you plan migrations before forced removal breaks workloads.

## Check Deprecation Warnings

Ceph surfaces deprecation warnings in the health output:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph health detail | grep -i deprecat
```

Review the Ceph log for deprecation notices:

```bash
kubectl -n rook-ceph logs deploy/rook-ceph-mon-a | grep -i deprecated
```

## Key Deprecated Features in Reef

### Legacy CephFS MDS Affinity

The old MDS affinity configuration using `mds_standby_for_name` is deprecated:

```bash
# Check if old affinity settings are in use
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph config dump | grep mds_standby
```

Migrate to the new `fs set standby_count_wanted` approach:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph fs set myfs standby_count_wanted 1
```

### RBD Mirroring Bootstrap Token Format

The old bootstrap token format for RBD mirroring is deprecated. Generate a new token:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- rbd mirror pool peer bootstrap create \
  --site-name primary mypool
```

### Legacy OSD Creation via ceph-disk

Any remaining `ceph-disk`-based OSDs should be migrated to `ceph-volume`:

```bash
# Check OSD deployment method
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd metadata | grep osd_objectstore
```

## Audit Your Configuration

Generate a full config dump to spot deprecated options:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph config dump > config-dump.txt
grep -E "(deprecated|legacy|old)" config-dump.txt
```

Remove deprecated config keys proactively:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph config rm global old_option_name
```

## Update Rook CRDs

Check your CephCluster and related CRDs for deprecated fields in the Rook API:

```bash
kubectl get cephcluster rook-ceph -n rook-ceph -o yaml | grep -i deprecated
```

Compare against the Rook v1.14+ CRD reference to identify removed fields.

## Summary

Ceph Reef deprecates several legacy features including old MDS affinity settings, bootstrap token formats, and ceph-disk OSD creation. Use `ceph health detail` and `ceph config dump` to audit your cluster, and proactively migrate deprecated configurations before upgrading to Squid to prevent service disruptions.
