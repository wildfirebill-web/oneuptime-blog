# How to Troubleshoot Ceph After OS Updates

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, Operating System, Upgrade, Troubleshooting

Description: Diagnose and resolve Ceph issues caused by OS updates including kernel regressions, library incompatibilities, systemd unit changes, and network driver issues.

---

OS updates are a common trigger for Ceph cluster issues. Kernel regressions, updated libraries, or changed service configurations can all destabilize a previously healthy cluster.

## Verify What Changed

Before investigating Ceph-specific problems, confirm what was updated:

```bash
# On RPM-based systems
rpm -qa --last | head -30

# On Debian/Ubuntu
grep " install\| upgrade" /var/log/dpkg.log | tail -30

# Check kernel version
uname -r
```

If the kernel changed, that is the first suspect for OSD crashes or network issues.

## Check for OSD Crashes After Reboot

After an OS update and reboot, check if all OSDs came back up:

```bash
ceph osd tree | grep -E "down|out"
ceph health detail

# Check specific OSD service
systemctl status ceph-osd@0
journalctl -u ceph-osd@0 --since "1 hour ago" -p err
```

## Kernel-Related OSD Issues

Some kernel versions have regressions affecting BlueStore's NVMe or io_uring support:

```bash
# Check dmesg for storage-related errors
dmesg | grep -iE "nvme|xfs|ext4|blkdev|error|WARN" | tail -30

# Check if the kernel module for your storage backend loaded correctly
lsmod | grep -E "nvme|dm_|rbd"
```

If you see blkdev errors after a kernel upgrade, booting the previous kernel is the fastest way to confirm the regression.

## Library Incompatibilities

Ceph links against glibc, OpenSSL, and other system libraries. Version mismatches can cause daemon startup failures:

```bash
# Check which libraries Ceph OSD links against
ldd /usr/bin/ceph-osd

# Verify no broken dependencies
ldd /usr/bin/ceph-osd | grep "not found"
```

If libraries are missing or mismatched, reinstalling the ceph-osd package resolves it:

```bash
# Ubuntu/Debian
apt-get install --reinstall ceph-osd

# RHEL/CentOS
dnf reinstall ceph-osd
```

## Systemd Unit Changes

OS updates occasionally change systemd behavior or reset overrides:

```bash
# Check if Ceph units are enabled
systemctl is-enabled ceph-osd@0
systemctl is-enabled ceph-mon@$(hostname)

# Re-enable if needed
systemctl enable ceph-osd@0
systemctl enable "ceph-mon@$(hostname)"
```

## Network Driver Regressions

Kernel updates sometimes change network driver behavior:

```bash
ethtool -i eth0
ethtool eth0 | grep -E "Speed|Duplex|Link"

# Check for interface errors
ip -s link show eth0 | grep -E "error|drop"
```

If the cluster public or cluster networks show elevated errors, test with a rollback to the previous kernel.

## Summary

After OS updates, Ceph issues most commonly stem from kernel regressions affecting storage or network drivers, library version mismatches, or systemd unit changes. Start by identifying exactly what was updated, then check OSD status and journal logs for startup failures. Booting the previous kernel is the quickest way to isolate a kernel regression.
