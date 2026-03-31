# How to Add New Nodes to a Running Rook-Ceph Cluster

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Node Management, Scaling, Kubernetes, Storage

Description: Learn how to safely add new storage nodes to a running Rook-Ceph cluster - covering node labeling, CephCluster configuration, OSD provisioning, and CRUSH map updates.

---

## Overview

Adding nodes to a Rook-Ceph cluster expands both capacity and redundancy. The process involves preparing the new nodes, updating the CephCluster specification, and verifying that OSDs are provisioned and data rebalances correctly.

## Step 1: Prepare the New Node

Ensure the new node meets Rook requirements:

```bash
# Verify the node is joined to the Kubernetes cluster
kubectl get node new-worker-4

# Check disk is available and empty
ssh new-worker-4 lsblk
ssh new-worker-4 sudo sgdisk --print /dev/sdb
```

Label the node for Ceph scheduling:

```bash
kubectl label node new-worker-4 role=storage-node
```

If using node affinity in the CephCluster, add the required labels:

```bash
kubectl label node new-worker-4 ceph-osd=enabled
kubectl label node new-worker-4 topology.kubernetes.io/zone=us-east-1a
```

## Step 2: Update the CephCluster Specification

Option A - Using `useAllNodes: true` (automatically picks up new nodes with the right labels):

```yaml
spec:
  storage:
    useAllNodes: true
    useAllDevices: true
    deviceFilter: "^sd[b-z]"
```

Option B - Explicit node list (add the new node entry):

```yaml
spec:
  storage:
    nodes:
    - name: "worker-1"
      devices:
      - name: "sdb"
    - name: "worker-2"
      devices:
      - name: "sdb"
    - name: "new-worker-4"    # add this
      devices:
      - name: "sdb"
```

Apply the updated spec:

```bash
kubectl -n rook-ceph apply -f ceph-cluster.yaml
```

## Step 3: Watch OSD Provisioning

The Rook operator detects the change and provisions new OSDs:

```bash
watch kubectl -n rook-ceph get pods -l app=rook-ceph-osd
```

You should see new OSD pods starting on `new-worker-4`. Check the operator logs:

```bash
kubectl -n rook-ceph logs deploy/rook-ceph-operator --tail=50 | grep -i "provision\|new-worker-4"
```

## Step 4: Verify OSD is In Cluster

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd tree
```

The new OSD(s) should appear under `new-worker-4` in the CRUSH tree with status `up` and `in`.

## Step 5: Verify Data Rebalancing

Ceph automatically rebalances data across the new OSDs:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph status
# Look for "misplaced" count decreasing over time

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd df tree
# Verify usage percentages are balancing
```

## Step 6: Update CRUSH Weights if Needed

If the new node has different disk sizes:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd crush reweight osd.9 2.0
```

## Summary

Adding nodes to a Rook-Ceph cluster requires labeling the node, updating the CephCluster spec, and verifying OSD provisioning and data rebalancing. Using `useAllNodes: true` with node selectors allows new nodes to be automatically discovered without manual CephCluster edits.
