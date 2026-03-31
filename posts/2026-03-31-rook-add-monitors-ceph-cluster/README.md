# How to Add New Monitors to a Ceph Cluster

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Monitor, Quorum, Administration

Description: Learn how to add new Ceph monitors to a Rook cluster to increase quorum size, improve fault tolerance, and support cluster expansion.

---

## Why Add More Monitors?

Ceph monitors (MONs) maintain the cluster map and manage distributed consensus using the Paxos algorithm. Quorum requires a majority of monitors to be available: a 3-MON cluster tolerates 1 failure, a 5-MON cluster tolerates 2 failures. Starting with 3 monitors is standard. Adding more is appropriate when expanding across availability zones or when you need higher fault tolerance. Always use odd numbers (3, 5) to maintain clear majority quorum.

## Checking Current Monitor Count

View current monitors and their status:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph mon stat

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph quorum_status --format json-pretty | python3 -m json.tool
```

## Adding Monitors via CephCluster CRD

In Rook, monitor count is controlled by the `CephCluster` spec. To increase from 3 to 5 monitors, edit the resource:

```bash
kubectl -n rook-ceph edit cephcluster rook-ceph
```

Update the `mon` section:

```yaml
spec:
  mon:
    count: 5
    allowMultiplePerNode: false
```

The Rook operator detects the change and adds new MON pods automatically. Watch the operator logs:

```bash
kubectl -n rook-ceph logs deploy/rook-ceph-operator -f | grep -i mon
```

## Pinning Monitors to Specific Nodes

To place new monitors on specific nodes, use placement configuration:

```yaml
spec:
  placement:
    mon:
      nodeAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
          nodeSelectorTerms:
          - matchExpressions:
            - key: node-role.kubernetes.io/monitor
              operator: In
              values:
              - "true"
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: rook-ceph-mon
```

Label the target nodes:

```bash
kubectl label node worker-1 node-role.kubernetes.io/monitor=true
kubectl label node worker-2 node-role.kubernetes.io/monitor=true
kubectl label node worker-3 node-role.kubernetes.io/monitor=true
```

## Verifying the New Monitors

After Rook adds the monitors, confirm quorum:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph mon dump

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph quorum_status
```

Check that all new MON pods are running:

```bash
kubectl -n rook-ceph get pods -l app=rook-ceph-mon
```

## PVC-Backed Monitors

For production clusters, use PVC-backed monitors for persistent storage:

```yaml
spec:
  mon:
    count: 5
    volumeClaimTemplate:
      spec:
        storageClassName: fast-storage
        resources:
          requests:
            storage: 10Gi
```

## Summary

Adding monitors in Rook is a declarative operation - increase `mon.count` in the `CephCluster` spec and the operator handles the rest. Always use odd numbers (3 or 5) to maintain quorum. Use node affinity and topology spread constraints to distribute monitors across failure domains for maximum availability.
