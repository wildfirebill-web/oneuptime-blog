# How to Troubleshoot Volume Mount Failures on Specific Nodes in Rook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Kubernetes, Troubleshooting, Mount, Node

Description: Diagnose volume mount failures that only occur on specific Kubernetes nodes in Rook-Ceph, including kernel module, CSI plugin, and network issues.

---

## Overview

When volumes mount successfully on most nodes but fail on specific ones, the issue is node-specific rather than a cluster-wide problem. This pattern points to missing kernel modules, CSI plugin failures on the node, or network configuration differences.

## Identifying Node-Specific Failures

First, identify which nodes are affected by checking failing pods and their placement:

```bash
kubectl get pods -A -o wide | grep -v Running | grep -v Completed
```

Note which node name appears in the `NODE` column for failing pods. Compare with healthy pods using the same PVC type.

## Checking CSI Plugin Pod on the Problem Node

Each node runs a CSI plugin pod. Find and inspect it:

```bash
kubectl get pod -n rook-ceph -o wide | grep csi-rbdplugin | grep <node-name>

kubectl logs -n rook-ceph <csi-rbdplugin-pod-name> -c csi-rbdplugin | \
  tail -100 | grep -iE "error|fail|warn"
```

## Checking Kernel Modules

RBD volumes require the `rbd` kernel module. CephFS volumes require `ceph` kernel module. Verify on the affected node:

```bash
# SSH to the node or use a privileged debug pod
kubectl debug node/<node-name> -it --image=ubuntu -- \
  bash -c "lsmod | grep -E 'rbd|ceph'"
```

If the module is missing:

```bash
modprobe rbd
# or
modprobe ceph
```

For the module to load persistently:

```bash
echo "rbd" > /etc/modules-load.d/rbd.conf
```

## Checking Kubelet Logs on the Node

Mount operations are coordinated by kubelet. Check kubelet logs on the affected node:

```bash
kubectl get events -n <namespace> \
  --field-selector involvedObject.name=<pod-name> | grep -i mount

# Or use journalctl via a debug pod
kubectl debug node/<node-name> -it --image=ubuntu -- \
  chroot /host journalctl -u kubelet | grep -i "mount\|ceph\|rbd" | tail -50
```

## Checking the CSI Plugin Registration

The CSI node plugin must be registered with kubelet on each node. Verify:

```bash
kubectl get csinode <node-name> -o yaml
```

Look for `rook-ceph.rbd.csi.ceph.com` and `rook-ceph.cephfs.csi.ceph.com` in the drivers list. If missing, the CSI plugin failed to register. Restart the plugin pod on that node:

```bash
kubectl delete pod -n rook-ceph <csi-rbdplugin-pod-on-node>
```

## Network Connectivity from the Node

The CSI plugin on the node needs to reach Ceph monitors. Test from the node directly:

```bash
kubectl debug node/<node-name> -it --image=busybox -- \
  sh -c "nc -zv <monitor-ip> 6789 && echo OK || echo FAIL"
```

Firewalls, node-specific routing, or NetworkPolicy differences can block monitor access on specific nodes while others work fine.

## Comparing Working vs. Problem Nodes

Diff the configurations between a working node and the problem node:

```bash
# Check kernel versions
kubectl get node <problem-node> -o jsonpath='{.status.nodeInfo.kernelVersion}'
kubectl get node <working-node> -o jsonpath='{.status.nodeInfo.kernelVersion}'

# Check OS image
kubectl get node -o custom-columns=NAME:.metadata.name,OS:.status.nodeInfo.osImage
```

Older kernel versions may lack required RBD features.

## Summary

Node-specific volume mount failures in Rook-Ceph are diagnosed by checking the CSI plugin pod on the affected node, verifying kernel module availability, inspecting CSI node registration, and testing monitor network connectivity. Most node-specific failures trace to missing kernel modules or CSI plugin registration issues that a pod restart resolves.
