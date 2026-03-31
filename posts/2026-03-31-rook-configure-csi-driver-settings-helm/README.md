# How to Configure CSI Driver Settings (RBD, CephFS, NFS) in Rook Helm

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Kubernetes, CSI, Helm, Storage

Description: Configure RBD, CephFS, and NFS CSI driver settings in the Rook-Ceph operator Helm chart to enable and tune persistent volume provisioning.

---

## Overview

Rook-Ceph deploys CSI drivers for three storage protocols: RBD (block storage), CephFS (shared filesystem), and NFS. Each driver is independently configurable through the operator Helm chart `csi` values section.

## Enabling and Disabling Drivers

By default, RBD and CephFS are enabled while NFS is disabled. Control this per driver:

```yaml
csi:
  enableRbdDriver: true
  enableCephfsDriver: true
  enableNFSDriver: false
```

To add NFS support, set `enableNFSDriver: true` and ensure NFS kernel modules are available on nodes.

## RBD Driver Settings

The RBD driver handles block storage PVCs. Key settings:

```yaml
csi:
  rbdFSGroupPolicy: File
  enableRBDSnapshotter: true
  enableOMAPGenerator: true
  rbdPluginUpdateStrategy: OnDelete
  rbdPluginUpdateStrategyMaxUnavailable: 1
```

`rbdFSGroupPolicy` controls how the CSI driver applies the pod's `fsGroup` to RBD volumes. `File` applies it recursively, `None` skips it.

## CephFS Driver Settings

CephFS handles shared filesystem PVCs and subvolumes:

```yaml
csi:
  enableCephfsSnapshotter: true
  cephFSFUSEClient: false
  enableCSIHostNetwork: false
  cephfsPluginUpdateStrategy: OnDelete
  cephfsPluginUpdateStrategyMaxUnavailable: 1
  fuseMountOptions: ""
  kernelMountOptions: "ms_mode=prefer-crc"
```

Setting `cephFSFUSEClient: true` forces the FUSE client instead of the kernel client - useful for environments where kernel modules are unavailable.

## NFS Driver Settings

When NFS is enabled:

```yaml
csi:
  enableNFSDriver: true
  nfsPluginUpdateStrategy: OnDelete
```

## Driver Registration Settings

CSI driver registration controls node-level integration:

```yaml
csi:
  registrar:
    image:
      tag: v2.9.0

  provisioner:
    image:
      tag: v3.7.0

  attacher:
    image:
      tag: v4.4.0

  resizer:
    image:
      tag: v1.9.0
```

## Applying Driver Configuration

```bash
helm upgrade rook-ceph rook-release/rook-ceph \
  --namespace rook-ceph \
  -f rook-csi-values.yaml
```

Verify all CSI pods are running after upgrade:

```bash
kubectl get pods -n rook-ceph | grep csi
```

Expected pods include `csi-rbdplugin-*`, `csi-rbdplugin-provisioner-*`, `csi-cephfsplugin-*`, and `csi-cephfsplugin-provisioner-*`.

## Summary

The Rook-Ceph Helm chart provides granular control over each CSI driver. Enable only the drivers you need, tune snapshot support, and adjust mount options per protocol. RBD handles block workloads, CephFS handles shared access, and NFS is available when Ganesha-based NFS exports are required.
