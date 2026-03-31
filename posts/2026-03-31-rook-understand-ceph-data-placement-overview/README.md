# How to Understand Ceph Data Placement Overview

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, CRUSH, Data Placement, Architecture

Description: Learn how Ceph uses CRUSH maps, placement groups, and pools to distribute data across OSDs for resilience and performance.

---

## Data Placement in Ceph

Ceph does not use a centralized directory to track where data lives. Instead, it uses a deterministic algorithm called CRUSH (Controlled Replication Under Scalable Hashing) to calculate the location of any object given the cluster state. This eliminates lookup bottlenecks and allows the cluster to scale without a central metadata server.

## The Data Placement Pipeline

When a client writes an object, Ceph:

1. Hashes the object name to compute its placement group (PG) ID
2. Looks up the PG in the CRUSH map to find which OSDs should store it
3. Writes the object to the primary OSD
4. The primary OSD replicates to the replica OSDs

The formula is:

```text
Object Name -> Hash -> PG ID -> CRUSH map -> OSD set
```

## Pools

Pools are the top-level logical containers. Each pool has:

- A replication factor or erasure coding profile
- A CRUSH rule that controls which part of the cluster hosts it
- A PG count

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd lspools
```

## Placement Groups

PGs are the intermediary between objects and OSDs. Each object belongs to one PG; each PG maps to a set of OSDs. The number of PGs determines data distribution granularity.

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph pg stat
```

View which OSDs a specific PG maps to:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph pg map 1.0
```

Output:

```text
osdmap e42 pg 1.0 (1.0) -> up [2,5,9] acting [2,5,9]
```

The `up` set is the target according to CRUSH; the `acting` set is the current set that may differ during recovery.

## CRUSH Hierarchy

The CRUSH map defines a hierarchy of bucket types (root, datacenter, rack, host, OSD) that represent physical topology. CRUSH uses this hierarchy to place replicas across different failure domains.

View the current CRUSH map:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd crush tree
```

## Device Classes

Ceph supports device classes (hdd, ssd, nvme) within the CRUSH hierarchy. You can pin specific pools to specific device classes:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: fast-pool
  namespace: rook-ceph
spec:
  replicated:
    size: 3
  deviceClass: ssd
```

## Data Distribution Balance

Check how evenly data is distributed across OSDs:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd df
```

The `VAR` column shows variance. Values significantly above 1.0 indicate uneven distribution, which can be corrected using the balancer module or CRUSH weight adjustments.

## Summary

Ceph data placement uses a pipeline from object to pool to placement group to OSD, driven by the CRUSH algorithm. Understanding this pipeline is essential for diagnosing imbalance, configuring failure domains, and optimizing performance. Use `ceph pg map`, `ceph osd crush tree`, and `ceph osd df` to inspect and validate your data placement configuration.
