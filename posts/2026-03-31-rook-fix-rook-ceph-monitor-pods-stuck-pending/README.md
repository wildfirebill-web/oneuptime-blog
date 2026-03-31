# How to Fix Rook-Ceph Monitor Pods Stuck in Pending

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Troubleshooting, Monitor, Kubernetes

Description: Learn how to fix Rook-Ceph monitor pods stuck in Pending state by resolving node affinity conflicts, storage class issues, resource constraints, and taint tolerations.

---

## Why Monitor Pods Get Stuck

Rook-Ceph monitor (MON) pods are critical for cluster quorum. They use PersistentVolumeClaims or hostPath storage and have node affinity rules to distribute across failure domains. When they cannot be scheduled, the cluster cannot form quorum and no data operations can proceed.

## Step 1: Identify the Pending Pod

```bash
kubectl -n rook-ceph get pods -l app=rook-ceph-mon

# Find which pods are pending
kubectl -n rook-ceph get pods -l app=rook-ceph-mon | grep Pending

# Describe the pending pod for scheduling reasons
kubectl -n rook-ceph describe pod rook-ceph-mon-a-<id>
```

Look for the `Events:` section at the bottom of the describe output. Common messages:

- `0/3 nodes are available: 3 Insufficient memory` - resource constraints
- `0/3 nodes are available: 3 node(s) didn't match node affinity` - affinity mismatch
- `0/3 nodes are available: 3 node(s) had taint...that the pod didn't tolerate` - taint issue

## Step 2: Check Node Resources

```bash
kubectl top nodes
kubectl describe node <node-name> | grep -E "Requests|Limits|Allocatable"
```

If nodes are running low on memory or CPU, free up resources by scaling down other workloads or adding nodes.

## Step 3: Check Node Labels for Affinity

Rook mon pods use node affinity based on labels. Verify nodes have the required labels:

```bash
kubectl get nodes --show-labels | grep -E "hostname|zone|region"
```

If the CephCluster spec specifies a placement policy, verify the labels match:

```yaml
spec:
  mon:
    count: 3
    allowMultiplePerNode: false
  placement:
    mon:
      nodeAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
          nodeSelectorTerms:
            - matchExpressions:
                - key: node-role.kubernetes.io/worker
                  operator: Exists
```

Ensure worker nodes have the label:

```bash
kubectl label node worker-node-1 node-role.kubernetes.io/worker=""
```

## Step 4: Check Node Taints

If nodes have taints that mon pods do not tolerate:

```bash
kubectl describe node worker-node-1 | grep Taints
```

Add tolerations in the CephCluster spec:

```yaml
spec:
  placement:
    mon:
      tolerations:
        - key: storage-node
          operator: Exists
          effect: NoSchedule
```

## Step 5: Check PVC Provisioning (if using PVC-based mons)

If monitors use PVCs for storage, the PVC must bind before the pod can start:

```bash
kubectl -n rook-ceph get pvc | grep mon

# If PVC is Pending, check the StorageClass
kubectl -n rook-ceph describe pvc rook-ceph-mon-a
```

Ensure the configured StorageClass exists and the provisioner is running:

```bash
kubectl get storageclass
```

## Step 6: Allow Multiple Mons Per Node (Last Resort)

If you have fewer nodes than desired mon count, allow multiple mons per node temporarily:

```yaml
spec:
  mon:
    count: 3
    allowMultiplePerNode: true
```

```bash
kubectl -n rook-ceph edit cephcluster rook-ceph
```

This reduces fault tolerance but unblocks cluster formation while you add more nodes.

## Verify Quorum After Fixing

Once all mon pods are Running, verify quorum is established:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash -c "
  ceph mon stat
  ceph quorum_status --format json | jq '.quorum_names'
"
```

All three monitor names should appear in the quorum list.

## Summary

Rook-Ceph monitor pods stuck in Pending are most commonly caused by insufficient node resources, node affinity mismatches due to missing labels, nodes with taints that mon pods do not tolerate, or PVC provisioning failures. Diagnosing through `kubectl describe pod` events pinpoints the cause, and fixes include adding node labels, tolerations, increasing resource capacity, or temporarily enabling multiple mons per node.
