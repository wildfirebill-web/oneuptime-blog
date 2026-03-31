# How to Set Pod Anti-Affinity Rules for Rook-Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Kubernetes, Placement, AntiAffinity, HighAvailability

Description: Configure pod anti-affinity rules in Rook-Ceph to prevent monitor, manager, and OSD pods from co-locating on the same node, protecting against single-node failures.

---

## Why Anti-Affinity Matters for Ceph

Ceph is designed to tolerate node failures, but only if its daemons are actually spread across nodes. Without anti-affinity rules, the Kubernetes scheduler can legally place multiple monitor pods on the same node. If that node fails, you lose quorum and the cluster becomes read-only.

Anti-affinity rules are your safety net against the scheduler making decisions that undermine Ceph's fault-tolerance model.

## Configuring Anti-Affinity in the CephCluster CRD

Rook exposes `placement` for each daemon type. The `podAntiAffinity` field maps directly to the Kubernetes Pod spec:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  placement:
    mon:
      podAntiAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
                - key: app
                  operator: In
                  values:
                    - rook-ceph-mon
            topologyKey: kubernetes.io/hostname
    mgr:
      podAntiAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
                - key: app
                  operator: In
                  values:
                    - rook-ceph-mgr
            topologyKey: kubernetes.io/hostname
    osd:
      podAntiAffinity:
        preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                  - key: app
                    operator: In
                    values:
                      - rook-ceph-osd
              topologyKey: kubernetes.io/hostname
```

Monitors and managers use `required` anti-affinity since placing two on one node is always wrong. OSDs use `preferred` because on small clusters you may have more OSDs than nodes.

## Required vs Preferred

Use `requiredDuringSchedulingIgnoredDuringExecution` when co-location is unacceptable:

```yaml
requiredDuringSchedulingIgnoredDuringExecution:
  - labelSelector:
      matchLabels:
        app: rook-ceph-mon
    topologyKey: kubernetes.io/hostname
```

Use `preferredDuringSchedulingIgnoredDuringExecution` when you want to spread pods but cannot guarantee enough nodes exist:

```yaml
preferredDuringSchedulingIgnoredDuringExecution:
  - weight: 100
    podAffinityTerm:
      labelSelector:
        matchLabels:
          app: rook-ceph-osd
      topologyKey: kubernetes.io/hostname
```

## Zone-Level Anti-Affinity

For multi-zone clusters, spread monitors across zones rather than just hosts:

```yaml
placement:
  mon:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app: rook-ceph-mon
          topologyKey: topology.kubernetes.io/zone
```

This ensures each monitor lands in a different availability zone, making the cluster resilient to zone-level outages.

## Verifying Anti-Affinity is Active

Describe a running monitor pod to confirm the rule is applied:

```bash
kubectl -n rook-ceph describe pod -l app=rook-ceph-mon | grep -A 20 "Anti-Affinity"
```

Check which nodes the monitors are running on:

```bash
kubectl -n rook-ceph get pod -l app=rook-ceph-mon -o wide
```

All monitors should show different values in the `NODE` column.

## Summary

Pod anti-affinity rules in Rook ensure that Ceph daemons honor the same fault-tolerance principles at the scheduling layer that they implement at the storage layer. Apply required anti-affinity for monitors and managers, and preferred anti-affinity for OSDs to keep your cluster resilient against node failures without over-constraining the scheduler.
