# How to Understand Placement Groups in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, PlacementGroup, Storage, Performance, Kubernetes

Description: Understand Ceph placement groups - the internal sharding mechanism that distributes objects across OSDs and controls recovery granularity.

---

## What Are Placement Groups

Placement Groups (PGs) are the internal sharding layer between objects and OSDs in Ceph. Rather than mapping each object directly to a specific OSD (which would require an enormous lookup table), Ceph first maps objects to a PG, then maps PGs to OSDs via CRUSH.

This two-level indirection makes it feasible to manage millions of objects efficiently. Each PG is a logical group of objects that share the same set of OSDs.

## How Objects Map to PGs

```text
hash(object_name) % pg_num = PG_ID
CRUSH(PG_ID) = [osd.4, osd.11, osd.19]
```

The hash is computed client-side, so no central service is required. Changing `pg_num` causes objects to remap across PGs and triggers data migration.

## PG States

Every PG is always in one or more states. Understanding states is essential for cluster diagnostics:

```bash
kubectl exec -it rook-ceph-tools -n rook-ceph -- ceph pg stat
```

Common states:

| State | Meaning |
|-------|---------|
| `active+clean` | Healthy - all replicas present |
| `active+degraded` | Serving I/O but missing replicas |
| `peering` | OSDs negotiating PG ownership |
| `stale` | No primary OSD reporting for this PG |
| `backfilling` | New OSDs receiving object copies |
| `recovering` | Rebuilding objects after OSD failure |
| `undersized` | Fewer active OSDs than required |

## Checking PG Health

```bash
# Summary of PG states
kubectl exec -it rook-ceph-tools -n rook-ceph -- ceph pg stat

# Find stuck PGs
kubectl exec -it rook-ceph-tools -n rook-ceph -- ceph pg dump_stuck

# Query a specific PG
kubectl exec -it rook-ceph-tools -n rook-ceph -- ceph pg 2.1e query
```

## Choosing the Right pg_num

Too few PGs cause uneven data distribution; too many create excessive overhead (each PG consumes ~150 KB of OSD memory). The general formula:

```text
Target PGs per OSD: 100-200
pg_num = (OSDs x target_pgs_per_osd) / replication_size
```

For 12 OSDs with 3x replication targeting 100 PGs per OSD:

```text
pg_num = (12 x 100) / 3 = 400
```

Round to the nearest power of 2: `512`.

Set PG count in a Rook pool:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: replicapool
  namespace: rook-ceph
spec:
  replicated:
    size: 3
  parameters:
    pg_num: "512"
    pgp_num: "512"
```

## PG Autoscaling

Ceph includes a PG autoscaler that adjusts `pg_num` based on pool usage. Enable it for a pool:

```bash
ceph osd pool set replicapool pg_autoscale_mode on
ceph osd pool autoscale-status
```

The autoscaler monitors pool sizes and adjusts PG counts to maintain the target PGs-per-OSD ratio without manual intervention.

## PG and Recovery Granularity

Recovery after an OSD failure happens at the PG level. Each PG independently recovers by copying objects from healthy replicas to new OSDs. More PGs mean finer-grained recovery and better parallelism, but also more overhead per PG.

```bash
# Watch recovery progress
kubectl exec -it rook-ceph-tools -n rook-ceph -- ceph -w | grep recovering
```

## Summary

Placement Groups are the fundamental distribution unit in Ceph, sitting between objects and OSDs. They enable efficient distributed hashing, fine-grained recovery parallelism, and manageable metadata overhead. Choosing the right `pg_num`, enabling autoscaling, and understanding PG states are core skills for operating healthy Rook-Ceph clusters at scale.
