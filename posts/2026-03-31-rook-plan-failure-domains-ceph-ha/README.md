# How to Plan Failure Domains for Ceph HA

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Failure Domain, High Availability, CRUSH, Storage

Description: Plan and configure Ceph failure domains using CRUSH rules in Rook to ensure data survives the loss of a host, rack, or availability zone.

---

Failure domains define the boundaries within which Ceph distributes replicas. Proper planning ensures that losing a host, rack, or data center zone does not result in data loss or unavailability.

## Understanding CRUSH Failure Domains

Ceph's CRUSH algorithm places replicas according to a hierarchy you define. Common failure domain levels include:

- `osd` - single disk (no redundancy for host failure)
- `host` - entire node failure (default in most deployments)
- `rack` - entire rack failure
- `zone` - availability zone failure

## Label Nodes for CRUSH Topology

Label Kubernetes nodes with topology information:

```bash
kubectl label node node1 topology.kubernetes.io/zone=zone-a
kubectl label node node2 topology.kubernetes.io/zone=zone-a
kubectl label node node3 topology.kubernetes.io/zone=zone-b
kubectl label node node4 topology.kubernetes.io/zone=zone-b
kubectl label node node5 topology.kubernetes.io/zone=zone-c
kubectl label node node6 topology.kubernetes.io/zone=zone-c
```

Add rack labels for rack-aware deployments:

```bash
kubectl label node node1 topology.rook.io/rack=rack-1
kubectl label node node2 topology.rook.io/rack=rack-1
kubectl label node node3 topology.rook.io/rack=rack-2
```

## Configure Failure Domains in CephBlockPool

Set the failure domain when creating pools:

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
    requireSafeReplicaSize: true
```

For zone-level protection:

```yaml
spec:
  failureDomain: zone
  replicated:
    size: 3
    requireSafeReplicaSize: true
```

## Configure the CephCluster for Topology Awareness

Enable topology awareness in the cluster spec:

```yaml
spec:
  placement:
    all:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: rook-ceph-osd
```

## Verify CRUSH Map

Inspect the CRUSH tree to confirm the hierarchy:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd tree
```

Check which pools are using which failure domains:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd dump | grep "^pool"
```

## Summary

Planning failure domains before deploying Ceph avoids hard-to-fix topology problems later. Labeling nodes with zone and rack information and referencing those labels in `CephBlockPool` failure domain settings ensures Rook places replicas across distinct failure boundaries. Use `requireSafeReplicaSize: true` to prevent writes when the failure domain constraint cannot be satisfied.
