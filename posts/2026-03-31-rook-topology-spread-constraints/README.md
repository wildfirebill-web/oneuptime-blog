# How to Configure Topology Spread Constraints in Rook-Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Kubernetes, Placement, TopologySpread, HighAvailability

Description: Apply Kubernetes topology spread constraints to Rook-Ceph daemon pods to enforce even distribution across zones, racks, or nodes for better fault tolerance.

---

## Topology Spread Constraints vs Anti-Affinity

Pod anti-affinity prevents co-location but does not enforce balance. If you have five nodes across two zones and three monitors, anti-affinity might place two monitors in one zone and one in the other.

Topology spread constraints give you fine-grained control: specify the maximum skew (imbalance) allowed between topology domains. A `maxSkew: 1` rule across three zones guarantees monitors are distributed one per zone when three zones are available.

## Configuring Topology Spread Constraints

Add `topologySpreadConstraints` under the `placement` section of your `CephCluster` CRD:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  placement:
    mon:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: rook-ceph-mon
    osd:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: rook-ceph-osd
```

## Understanding the Key Fields

`maxSkew` defines the maximum allowed difference in pod count between the most-loaded and least-loaded domain. A value of `1` means if one zone has 2 monitors, no zone can have 0.

`whenUnsatisfiable` controls the behavior when the constraint cannot be met:
- `DoNotSchedule` - blocks scheduling (strict enforcement)
- `ScheduleAnyway` - schedules but penalizes violating domains (soft enforcement)

Use `DoNotSchedule` for monitors and managers where distribution is critical, and `ScheduleAnyway` for OSDs to avoid blocking cluster expansion.

## Combining Constraints

Stack multiple constraints to enforce distribution at multiple topology levels:

```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: rook-ceph-mon
  - maxSkew: 1
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: ScheduleAnyway
    labelSelector:
      matchLabels:
        app: rook-ceph-mon
```

This places monitors evenly across zones first (required), then across hosts within each zone (preferred).

## Applying to MDS and RGW Components

CephFilesystem and CephObjectStore components also support topology spread under their own placement sections:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephFilesystem
metadata:
  name: myfs
  namespace: rook-ceph
spec:
  metadataServer:
    activeCount: 1
    activeStandby: true
    placement:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: rook-ceph-mds
```

## Verifying Spread

Check how pods are distributed across zones:

```bash
kubectl -n rook-ceph get pod -l app=rook-ceph-mon \
  -o custom-columns='NAME:.metadata.name,NODE:.spec.nodeName,ZONE:.metadata.labels.topology\.kubernetes\.io/zone'
```

## Summary

Topology spread constraints in Rook provide declarative guarantees about how Ceph daemon pods are distributed across your infrastructure. They complement anti-affinity rules by enforcing balance rather than just separation, ensuring your Rook cluster remains resilient against zone or rack-level failures in multi-datacenter Kubernetes deployments.
