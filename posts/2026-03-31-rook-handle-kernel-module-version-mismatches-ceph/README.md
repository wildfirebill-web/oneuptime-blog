# How to Handle Kernel Module Version Mismatches with Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, Kernel, Module, Version, Compatibility, Upgrade

Description: Resolve version mismatches between the Linux kernel's Ceph modules and the userspace Ceph packages to restore connectivity and avoid protocol errors.

---

## Understanding Version Mismatches

Ceph has two clients that must be kept compatible:

1. **Kernel modules** (`rbd.ko`, `ceph.ko`) - version tied to the running kernel
2. **Userspace packages** (`ceph-common`, `rbd-nbd`, `ceph-fuse`) - version from the Ceph package repo

Mismatches occur when:
- The kernel is updated without updating Ceph packages
- Ceph packages are updated but the kernel is not upgraded
- DKMS-built modules fall out of sync after a kernel update

## Detecting a Mismatch

Check kernel module version:

```bash
modinfo rbd | grep -E "version|srcversion"
modinfo ceph | grep -E "version|srcversion"
```

Check userspace Ceph version:

```bash
ceph --version
rbd --version
```

Check negotiated protocol version during mount:

```bash
dmesg | grep -i "ceph.*version\|protocol"
```

## Protocol Version Negotiation

Ceph protocol includes version negotiation, so minor mismatches usually work. However:

- Kernel client `rbd.ko` from kernel 4.x connecting to Ceph Quincy (17.x) may lack new feature bits
- Old kernel clients may not support required MDS features in newer CephFS

```bash
# When mounting CephFS, look for feature negotiation in dmesg
dmesg | grep "ceph_mdsmap_decode\|feature"
```

## Resolving: Upgrade the Kernel

The cleanest fix when userspace Ceph is newer:

```bash
# Ubuntu - install latest HWE kernel
sudo apt-get install linux-generic-hwe-22.04
sudo reboot
uname -r  # verify
```

## Resolving: Use rbd-nbd Instead of Kernel RBD

`rbd-nbd` uses the userspace RBD library instead of the kernel module, bypassing the kernel version constraint:

```bash
sudo apt-get install rbd-nbd
rbd-nbd map mypool/myimage
rbd-nbd list-mapped
```

## Resolving: Use ceph-fuse Instead of Kernel CephFS

For CephFS feature mismatches:

```bash
sudo apt-get install ceph-fuse
sudo ceph-fuse /mnt/cephfs -n client.admin
```

## Disabling Unsupported RBD Features

If the kernel client does not support features enabled on an image:

```bash
# List image features
rbd info mypool/myimage | grep features

# Disable problematic features
rbd feature disable mypool/myimage object-map fast-diff deep-flatten
```

## Verifying DKMS Module Integrity

If using DKMS-built Ceph modules:

```bash
dkms status
dkms autoinstall -k $(uname -r)
```

If the module is outdated:

```bash
dkms remove -m ceph -v <old-version> --all
dkms install -m ceph -v <new-version> -k $(uname -r)
```

## Summary

Handle kernel/userspace Ceph version mismatches by upgrading the kernel for long-term resolution, or by switching to userspace alternatives (`rbd-nbd` for block, `ceph-fuse` for filesystems) as an immediate workaround. Disable RBD image features incompatible with older kernel clients using `rbd feature disable`, and use `dkms autoinstall` to rebuild any DKMS modules after kernel updates.
