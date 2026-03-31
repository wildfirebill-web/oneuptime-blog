# How to Check Kubelet Logs for Mount Failures in Rook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Kubernetes, Kubelet, Mount, Troubleshooting

Description: Access and interpret kubelet logs to diagnose volume mount failures in Rook-Ceph, including CSI stage errors and device attach problems.

---

## Overview

When a pod fails to start because a Rook-Ceph volume cannot be mounted, Kubernetes events provide limited information. The full mount error chain lives in kubelet logs on the node where the pod was scheduled. Accessing kubelet logs is essential for diagnosing CSI NodeStageVolume and NodePublishVolume failures.

## Finding the Affected Node

First, identify which node is hosting the failing pod:

```bash
kubectl get pod -n <namespace> <pod-name> -o wide
```

Note the `NODE` column. All kubelet logs for this pod will be on that node.

## Accessing Kubelet Logs via Node Debug Pod

The most practical way to access kubelet logs in Kubernetes:

```bash
kubectl debug node/<node-name> -it \
  --image=ubuntu \
  -- chroot /host journalctl -u kubelet -n 200
```

Filter for volume-related entries:

```bash
kubectl debug node/<node-name> -it \
  --image=ubuntu \
  -- chroot /host journalctl -u kubelet | \
  grep -E "mount|ceph|rbd|volume|csi" | tail -100
```

## Accessing Kubelet Logs via SSH

If you can SSH to the node directly:

```bash
journalctl -u kubelet -f | grep -E "mount|csi|rbd|ceph"
```

For a time window around the failure:

```bash
journalctl -u kubelet \
  --since "2026-03-31 10:00:00" \
  --until "2026-03-31 10:30:00" | \
  grep -E "mount|volume|error"
```

## Interpreting Common Kubelet Mount Errors

### NodeStageVolume Failed

```text
kubelet: volume.go:xxx]: error while running "NodeStageVolume":
  rpc error: code = Internal desc = failed to stage...
```

This means the CSI driver's NodeStageVolume RPC failed. The CSI driver was called but returned an error. Check the CSI node plugin log on this node:

```bash
kubectl logs -n rook-ceph <csi-rbdplugin-pod-on-node> -c csi-rbdplugin
```

### rbd map Failed

```text
rbd: sysfs write failed
In some cases useful info is found in syslog - try "dmesg | tail".
```

The kernel RBD module could not map the image. Check dmesg on the node:

```bash
kubectl debug node/<node-name> -it \
  --image=ubuntu \
  -- chroot /host dmesg | grep -iE "rbd|ceph|error" | tail -30
```

### Mount.ceph Failed

```text
mount: /var/lib/kubelet/pods/.../volume-subpaths/...:
  can't read superblock
```

This indicates a CephFS kernel client failure. Verify the monitor addresses are correct and the keyring has read permission on the filesystem.

## Checking Kubelet CSI Directory

Kubelet maintains a staging area for CSI volumes:

```bash
kubectl debug node/<node-name> -it \
  --image=ubuntu \
  -- ls /host/var/lib/kubelet/plugins/kubernetes.io/csi/
```

Stale mount points in this directory can block new mounts. Clean stale entries:

```bash
# Only if the volume is truly orphaned
umount /var/lib/kubelet/plugins/.../globalmount
rm -rf /var/lib/kubelet/plugins/kubernetes.io/csi/...
```

## Correlating Events with Kubelet Logs

Use Kubernetes events alongside kubelet logs for full context:

```bash
kubectl describe pod -n <namespace> <pod-name> | grep -A20 Events
```

The event timestamp helps you narrow the kubelet log time window.

## Summary

Kubelet logs are the authoritative source for volume mount failure details in Rook-Ceph. Access them via `kubectl debug node` or SSH, filter for CSI and mount keywords, and interpret NodeStageVolume errors by cross-referencing with CSI node plugin logs on the same node. Combined with Kubernetes pod events, kubelet logs provide the complete picture of every mount attempt.
