# How to Check Kernel Requirements for Rook-Ceph (RBD and CephFS Modules)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Kernel, RBD, CephFS, Linux Requirements

Description: Learn the Linux kernel module requirements for Rook-Ceph, how to verify RBD and CephFS module availability, and how to load them on your nodes.

---

## Why Kernel Modules Matter for Rook-Ceph

Rook-Ceph uses two Linux kernel modules on worker nodes that mount Ceph storage:

- **`rbd`** - Required for mounting Ceph block devices (RBD) as Kubernetes volumes
- **`ceph`** - Required for mounting CephFS shared filesystems

If these modules are missing or not loadable, CSI volume operations will fail. Pods that request PVCs backed by Rook-Ceph will be stuck in `Pending` or `ContainerCreating` state.

## Minimum Kernel Version Requirements

| Feature | Minimum Kernel |
|---------|----------------|
| RBD block storage | 3.10+ (recommended 4.17+) |
| CephFS kernel client | 3.10+ (recommended 4.17+) |
| RBD fast-diff feature | 4.2+ |
| RBD object-map feature | 4.9+ |
| RBD data encryption | 5.4+ |

Rook recommends kernel 4.17 or later for full feature support.

## Checking the Kernel Version on Nodes

```bash
# Check kernel version on a node
kubectl get nodes -o wide

# Or check directly on the node
uname -r
```

To check all nodes at once:

```bash
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.nodeInfo.kernelVersion}{"\n"}{end}'
```

## Verifying RBD Module Availability

```bash
# Check if the rbd module is loaded
lsmod | grep rbd

# Check if the module is available (even if not loaded)
modinfo rbd

# Load the rbd module
sudo modprobe rbd

# Verify it loaded
lsmod | grep rbd
```

Expected output after loading:

```text
rbd                    98304  0
libceph               335872  1 rbd
```

## Verifying CephFS Module Availability

```bash
# Check if the ceph kernel module is loaded
lsmod | grep ceph

# Check if it is available
modinfo ceph

# Load the ceph module
sudo modprobe ceph

# Verify
lsmod | grep ceph
```

Expected output:

```text
ceph                  458752  0
libceph               335872  2 rbd,ceph
fscache               131072  1 ceph
```

## Making Modules Load at Boot

To ensure modules load automatically on every boot:

```bash
# Add to /etc/modules-load.d/rook-ceph.conf
echo "rbd" | sudo tee /etc/modules-load.d/rook-ceph.conf
echo "ceph" | sudo tee -a /etc/modules-load.d/rook-ceph.conf

# Verify the file
cat /etc/modules-load.d/rook-ceph.conf
```

## Checking from Within a DaemonSet

Rook provides a way to verify kernel modules from within a pod using a privileged DaemonSet. This is useful when you cannot SSH to nodes directly:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kernel-check
  namespace: rook-ceph
spec:
  selector:
    matchLabels:
      app: kernel-check
  template:
    metadata:
      labels:
        app: kernel-check
    spec:
      hostPID: true
      containers:
        - name: kernel-check
          image: ubuntu:22.04
          command: ["/bin/bash", "-c", "modprobe rbd && modprobe ceph && echo 'Modules OK' && sleep 3600"]
          securityContext:
            privileged: true
```

```bash
kubectl apply -f kernel-check.yaml
kubectl -n rook-ceph logs -l app=kernel-check
```

## Troubleshooting Missing Kernel Modules

If `modinfo rbd` returns `modinfo: ERROR: Module rbd not found`, the kernel was compiled without RBD support. This is common with some minimal or hardened kernels.

```bash
# Check if kernel was compiled with RBD support
grep CONFIG_BLK_DEV_RBD /boot/config-$(uname -r)

# CONFIG_BLK_DEV_RBD=m  means it is a loadable module (good)
# CONFIG_BLK_DEV_RBD=y  means it is built-in (also good)
# Not present           means your kernel does not support RBD
```

If RBD is not supported, you need to either install a different kernel package or rebuild the kernel with `CONFIG_BLK_DEV_RBD=m`.

On Ubuntu:

```bash
# Install kernel with RBD support
sudo apt-get install linux-image-generic
sudo reboot
```

## Summary

Rook-Ceph requires the `rbd` and `ceph` kernel modules on every node that mounts Ceph volumes. Verify kernel version meets the minimum requirements, check module availability with `modinfo`, load them with `modprobe`, and ensure they persist across reboots via `/etc/modules-load.d/`. If modules are absent, install a distribution-provided generic kernel or use a privileged DaemonSet to load modules across all nodes simultaneously.
