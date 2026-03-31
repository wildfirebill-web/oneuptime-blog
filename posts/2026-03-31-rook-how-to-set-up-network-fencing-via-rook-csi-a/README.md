# How to Set Up Network Fencing via Rook CSI-Addons

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Csi-Addons, Network Fencing, Kubernetes, High Availability

Description: Learn how to configure network fencing via Rook CSI-Addons to block node access to Ceph storage during node failures for safe failover in Kubernetes.

---

## Overview

Network fencing is a mechanism used during node failures to prevent a failed or partitioned node from continuing to write to shared storage, avoiding data corruption. Rook CSI-Addons provides a `NetworkFence` CRD that allows Kubernetes operators or failover controllers to fence a node's network access to the Ceph cluster.

## Prerequisites

Install the CSI-Addons operator in your cluster:

```bash
kubectl apply -f https://github.com/csi-addons/kubernetes-csi-addons/releases/latest/download/crds.yaml
kubectl apply -f https://github.com/csi-addons/kubernetes-csi-addons/releases/latest/download/rbac.yaml
kubectl apply -f https://github.com/csi-addons/kubernetes-csi-addons/releases/latest/download/setup-controller.yaml
```

Verify the CSI-Addons controller is running:

```bash
kubectl -n csi-addons-system get pods
```

## Understanding NetworkFence

The `NetworkFence` CRD allows you to block CIDR ranges from accessing the Ceph cluster. When applied:
- Ceph OSD daemons reject connections from the specified IP ranges
- Any active I/O from those IPs is interrupted
- The node can no longer corrupt data even if the OS is in a degraded state

## Creating a NetworkFence

Create a NetworkFence to block a failed node:

```yaml
apiVersion: csiaddons.openshift.io/v1alpha1
kind: NetworkFence
metadata:
  name: fence-node-1
  namespace: rook-ceph
spec:
  driver: rook-ceph.rbd.csi.ceph.com
  fenceState: Fenced
  cidr:
    - 192.168.1.10/32
  secret:
    name: rook-csi-rbd-provisioner
    namespace: rook-ceph
```

Apply the fence:

```bash
kubectl apply -f network-fence.yaml
```

## Verifying the Fence is Active

Check the NetworkFence status:

```bash
kubectl -n rook-ceph get networkfence fence-node-1
```

Expected output:

```text
NAME           DRIVER                          FENCESTATE   AGE
fence-node-1   rook-ceph.rbd.csi.ceph.com     Fenced       30s
```

Verify the Ceph OSD blacklist:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd blocklist ls
```

Output showing the fenced IP:

```text
listed 1 entries.
192.168.1.10:0/0 2026-03-31T11:00:00.000000+0000
```

## Unfencing a Node

After the node is recovered, remove the fence:

```yaml
spec:
  fenceState: Unfenced
```

Apply the update:

```bash
kubectl -n rook-ceph patch networkfence fence-node-1 \
  --type merge \
  -p '{"spec":{"fenceState":"Unfenced"}}'
```

Verify the blocklist entry is removed:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd blocklist ls
```

## Integration with Failover Controllers

Network fencing is commonly used with storage-aware failover controllers like `openshift-dr` or custom operators. These controllers:

1. Detect node failure through heartbeat loss
2. Create a NetworkFence resource to block the failed node
3. Trigger pod failover to healthy nodes
4. Remove the NetworkFence after node recovery

Example automated fencing workflow using kubectl:

```bash
#!/bin/bash
FAILED_NODE_IP=$1

# Apply fence
cat <<EOF | kubectl apply -f -
apiVersion: csiaddons.openshift.io/v1alpha1
kind: NetworkFence
metadata:
  name: fence-${FAILED_NODE_IP//./-}
  namespace: rook-ceph
spec:
  driver: rook-ceph.rbd.csi.ceph.com
  fenceState: Fenced
  cidr:
    - ${FAILED_NODE_IP}/32
  secret:
    name: rook-csi-rbd-provisioner
    namespace: rook-ceph
EOF

echo "Node $FAILED_NODE_IP has been fenced"
```

## Summary

Network fencing via Rook CSI-Addons provides a mechanism to block a node's access to Ceph storage during failure scenarios, preventing data corruption from split-brain situations. The `NetworkFence` CRD adds the node's IP to Ceph's OSD blocklist, effectively cutting off its storage access. This is a critical component of high-availability storage architectures where safe, automated failover is required.
