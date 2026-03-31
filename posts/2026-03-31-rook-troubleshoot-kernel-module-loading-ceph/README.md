# How to Troubleshoot Kernel Module Loading Issues for Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, Kernel, Module, Troubleshooting, RBD, CephFS

Description: Diagnose and fix common issues when loading the rbd or ceph kernel modules, including missing modules, version mismatches, and dependency errors.

---

## Common Module Loading Errors

Kernel module loading failures manifest in several ways:

- `modprobe: FATAL: Module rbd not found`
- `modprobe: ERROR: could not insert 'rbd'`
- Module loads but device mapping fails
- `ceph.ko` version mismatch with running kernel

## Error: Module Not Found

```bash
modprobe rbd
# modprobe: FATAL: Module rbd not found in directory /lib/modules/5.15.0-91-generic
```

**Diagnosis:**

```bash
# Check if module file exists
find /lib/modules/$(uname -r) -name "rbd.ko*"

# Check kernel config
grep CONFIG_BLK_DEV_RBD /boot/config-$(uname -r)
```

**Fixes:**

```bash
# Install kernel extras package (Ubuntu/Debian)
sudo apt-get install linux-modules-extra-$(uname -r)

# RHEL/CentOS
sudo yum install kernel-modules-extra
```

## Error: Could Not Insert Module (Dependency)

```bash
modprobe rbd
# modprobe: ERROR: could not insert 'rbd': Unknown symbol in module
```

**Diagnosis:**

```bash
# Check dependencies
modinfo rbd | grep depends
# Typically: depends: libceph

# Load the dependency manually
modprobe libceph
dmesg | tail -20
```

**Fix:**

```bash
sudo modprobe libceph && sudo modprobe rbd
```

## Module Loads But Mapping Fails

If `modprobe rbd` succeeds but `rbd device map` fails:

```bash
# Check dmesg for kernel errors
dmesg | grep -i rbd | tail -30
```

Common causes:
- Network connectivity to Ceph monitors
- Authentication failure (wrong keyring)
- Feature incompatibility (see `rbd feature disable`)

## Checking Module Version vs. Kernel

```bash
modinfo ceph | grep -E "version|srcversion"
uname -r
```

They should correspond to the same kernel version.

## Locked Module (Secure Boot)

On systems with Secure Boot, unsigned modules will fail to load:

```bash
dmesg | grep "Lockdown\|prohibited"
```

Check Secure Boot status:

```bash
mokutil --sb-state
```

If Secure Boot is active, you must either disable it, enroll the module signing key, or use a distribution-signed module.

## Rebuilding Modules with DKMS

If you compiled a custom kernel or Ceph modules:

```bash
# Check DKMS status
dkms status

# Rebuild for the current kernel
dkms autoinstall
```

## Forcing a Module Reload

```bash
sudo rmmod rbd
sudo modprobe rbd
dmesg | tail -10
```

## Summary

Module loading issues are most commonly caused by missing `linux-modules-extra` packages, unresolved `libceph` dependencies, or Secure Boot restrictions. Use `dmesg`, `modinfo`, and `find /lib/modules` to diagnose, and install the appropriate kernel extras package as the first fix. For Secure Boot environments, verify module signing before blaming the module itself.
