# How to Set Failure Domain for Block Pools in Rook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Failure Domain, Block Storage, CRUSH, Kubernetes, High Availability

Description: Set the failure domain for Rook-Ceph block pools to control where replicas are placed across OSDs, hosts, racks, or zones for the right level of fault tolerance.

---

## What Is a Failure Domain

A failure domain defines the unit of failure that Ceph protects against. When you set `failureDomain: host`, Ceph ensures that no two replicas of the same object reside on the same host. If that host goes down, the remaining replicas are on different hosts and are immediately accessible. Choosing the right failure domain is a fundamental decision that directly affects cluster availability and the minimum hardware required.

## Available Failure Domains

Ceph CRUSH supports several hierarchical levels out of the box:

| Failure Domain | Description |
|---|---|
| `osd` | Each replica on a different OSD (least protection, most flexibility) |
| `host` | Each replica on a different host (standard production setting) |
| `rack` | Each replica in a different rack |
| `row` | Each replica in a different row |
| `datacenter` | Each replica in a different datacenter |
| `zone` | Each replica in a different zone (cloud or custom CRUSH topology) |
| `region` | Each replica in a different region |

## Setting failureDomain in CephBlockPool

The failure domain is set directly on the `CephBlockPool` spec:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: replicapool
  namespace: rook-ceph
spec:
  failureDomain: host
  replicated:
    size: 3
```

For a zone-level failure domain on a multi-zone cluster:

```yaml
spec:
  failureDomain: zone
  replicated:
    size: 3
```

## Hardware Requirements Per Failure Domain

The failure domain setting determines the minimum cluster size:

- `failureDomain: host` with `size: 3` requires at least three hosts with OSDs.
- `failureDomain: rack` with `size: 3` requires at least three racks.
- `failureDomain: zone` with `size: 3` requires at least three zones configured in the CRUSH map.

Verify your CRUSH topology:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd tree
```

## Custom CRUSH Topology for Zones

If you want zone-level failure domains, you must first define the topology in the CRUSH map. With Rook, you can use the `topology` labels on Kubernetes nodes and configure the CephCluster to use them:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  storage:
    useAllNodes: true
    useAllDevices: true
  placement:
    all:
      nodeAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
          nodeSelectorTerms:
            - matchExpressions:
                - key: topology.kubernetes.io/zone
                  operator: In
                  values:
                    - zone-a
                    - zone-b
                    - zone-c
```

Rook reads the `topology.kubernetes.io/zone` label from each node and populates the CRUSH map accordingly.

## Changing the Failure Domain on an Existing Pool

Changing the failure domain on a pool with data triggers a full backfill as Ceph remaps PGs to satisfy the new CRUSH rule. On large pools this is disruptive:

```bash
# Check current PG state before changing
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph pg stat

# Apply the updated CephBlockPool manifest
kubectl apply -f updated-pool.yaml

# Monitor recovery
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  watch ceph status
```

## Verifying the CRUSH Rule

After applying the pool spec, check which CRUSH rule was created:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd crush rule dump replicapool
```

The rule output shows the failure domain in the `step chooseleaf` line, confirming the setting took effect.

## Summary

The `failureDomain` field in a CephBlockPool spec controls which CRUSH bucket level separates replicas. Use `host` for most clusters, `rack` when hosts are distributed across physical racks, and `zone` for multi-zone deployments. Match the failure domain to the actual infrastructure topology to ensure replicas are genuinely independent and the cluster can survive realistic failure scenarios.
