# How to Use Overlay Storage Driver for Best Performance in Podman

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Overlay, Storage Driver, Performance, Filesystem, Linux, Containers, DevOps

Description: A deep dive into configuring and optimizing the overlay storage driver in Podman for maximum container filesystem performance, including rootless setup, kernel requirements, and tuning options.

---

> The overlay storage driver is the fastest and most efficient option for Podman container storage. Properly configured, it delivers near-native filesystem performance with minimal overhead.

The storage driver determines how Podman manages image layers and container filesystems. It affects every operation from pulling images to reading files inside a running container. The overlay driver uses the Linux kernel's OverlayFS to stack filesystem layers efficiently, providing copy-on-write semantics with excellent performance. This guide covers how to configure, optimize, and troubleshoot the overlay driver for both rootful and rootless Podman deployments.

---

## How OverlayFS Works

OverlayFS stacks multiple directories into a single unified view:

```text
Container View (merged)
├── /app/server          (from upper layer - container writes)
├── /usr/bin/curl        (from lower layer 2 - image layer)
├── /etc/config.yaml     (from lower layer 1 - base image)
└── /bin/sh              (from lower layer 0 - base image)

Upper Layer (read-write) ─── Container-specific changes
Lower Layer 2 (read-only) ── Application image layer
Lower Layer 1 (read-only) ── Dependencies layer
Lower Layer 0 (read-only) ── Base OS layer
```

When a container reads a file, OverlayFS finds it in the highest layer where it exists. When a container writes to a file from a lower layer, OverlayFS copies it to the upper layer first (copy-on-write). This means reads are fast because there is no data copying, and writes only incur overhead on the first modification.

---

## Verify and Enable the Overlay Driver

Check your current storage driver and switch to overlay:

```bash
# Check current driver
podman info --format '{{.Store.GraphDriverName}}'

# If not overlay, reset storage and reconfigure
podman system reset

# Configure overlay in storage.conf
# Rootful: /etc/containers/storage.conf
# Rootless: ~/.config/containers/storage.conf
```

```toml
# storage.conf
[storage]
driver = "overlay"
graphroot = "/var/lib/containers/storage"
runroot = "/run/containers/storage"

[storage.options.overlay]
# Overlay-specific options go here
```

After changing the driver, verify:

```bash
podman info --format '{{.Store.GraphDriverName}}'
# Output: overlay
```

---

## Kernel Requirements

The overlay driver requires specific kernel features:

```bash
# Check kernel version (4.0+ for basic overlay, 5.11+ for rootless)
uname -r

# Verify OverlayFS is available
grep overlay /proc/filesystems
# Output: nodev  overlay

# Check if the overlay module is loaded
lsmod | grep overlay

# Load it if missing
sudo modprobe overlay

# Make it persistent
echo "overlay" | sudo tee /etc/modules-load.d/overlay.conf
```

For rootless Podman without kernel 5.11+, you need fuse-overlayfs:

```bash
# Install fuse-overlayfs
# Debian/Ubuntu
sudo apt-get install -y fuse-overlayfs

# Fedora/RHEL
sudo dnf install -y fuse-overlayfs

# Verify installation
fuse-overlayfs --version

# Configure in storage.conf for rootless
# ~/.config/containers/storage.conf
```

```toml
[storage.options.overlay]
mount_program = "/usr/bin/fuse-overlayfs"
```

---

## Filesystem Requirements

The overlay driver requires specific filesystem features on the host:

```bash
# Check filesystem type of your storage location
df -T /var/lib/containers/storage

# XFS requires ftype=1 (d_type support)
xfs_info /var/lib/containers/storage | grep ftype
# Must show: ftype=1

# Create XFS with ftype=1 if needed
sudo mkfs.xfs -n ftype=1 /dev/sdX

# Ext4 supports overlay natively
# Btrfs and ZFS have their own overlay-like drivers
```

If your XFS filesystem has `ftype=0`, you must recreate it. There is no way to change this on an existing filesystem:

```bash
# Check and fix XFS ftype
xfs_info /mnt/data | grep ftype

# If ftype=0, backup data and recreate
sudo umount /mnt/data
sudo mkfs.xfs -n ftype=1 -f /dev/sdX
sudo mount /dev/sdX /mnt/data
# Restore data
```

---

## Enable Metacopy for Faster Copy-on-Write

Metacopy is an overlay optimization that copies only file metadata during copy-up, deferring data copy until the file is actually read or written:

```toml
# storage.conf
[storage.options.overlay]
mountopt = "nodev,metacopy=on"
```

Verify metacopy is active:

```bash
# Check if metacopy is enabled in the kernel
cat /sys/module/overlay/parameters/metacopy
# "Y" means enabled

# Enable metacopy at kernel level
echo "Y" | sudo tee /sys/module/overlay/parameters/metacopy

# Make persistent via kernel parameter
# Add to /etc/default/grub: overlay.metacopy=Y
```

Metacopy significantly speeds up container creation because large files in lower layers are not copied until they are actually modified. This is especially beneficial for images with large static assets.

---

## Configure Overlay Mount Options

Fine-tune overlay mount options for your workload:

```toml
# storage.conf
[storage.options.overlay]
# Recommended mount options
mountopt = "nodev,metacopy=on"

# For volatile workloads (faster, but crash-unsafe)
# mountopt = "nodev,metacopy=on,volatile"

# For redirect_dir (required for some rename operations)
# mountopt = "nodev,metacopy=on,redirect_dir=on"
```

The `volatile` option disables fsync on the upper layer, improving write performance at the cost of crash consistency. Use it for ephemeral workloads like CI/CD builds:

```bash
# Run a build container with volatile overlay
podman run --rm \
  --storage-opt "overlay.mountopt=volatile" \
  build-image make build
```

---

## Optimize Layer Management

The number and size of layers affect overlay performance. Optimize your image layers:

```bash
# Check layer count for an image
podman image inspect your-image --format '{{len .RootFS.Layers}}'

# View layer sizes
podman history your-image --format "table {{.Size}}\t{{.CreatedBy}}"

# Squash layers to reduce overhead
podman build --squash -t your-image:optimized .
```

Fewer layers mean faster container creation and less overhead for file lookups. However, this trades off against build cache efficiency:

```bash
# Compare performance: many layers vs squashed
echo "--- Many Layers ---"
time podman create --name test1 your-image:many-layers true
podman rm test1

echo "--- Squashed ---"
time podman create --name test2 your-image:squashed true
podman rm test2
```

---

## Rootless Overlay Configuration

Rootless Podman has specific overlay requirements:

```toml
# ~/.config/containers/storage.conf
[storage]
driver = "overlay"
graphroot = "/home/user/.local/share/containers/storage"

[storage.options]
# UID/GID mapping size
size = 65536

[storage.options.overlay]
# For kernel 5.11+: native overlay (best performance)
mount_program = ""

# For older kernels: fuse-overlayfs
# mount_program = "/usr/bin/fuse-overlayfs"
```

Ensure subuid/subgid are configured:

```bash
# Check configuration
grep $(whoami) /etc/subuid /etc/subgid

# Add if missing
sudo usermod --add-subuids 100000-165535 --add-subgids 100000-165535 $(whoami)

# Apply changes
podman system migrate
```

Compare rootless overlay performance:

```bash
# Benchmark native overlay vs fuse-overlayfs
echo "--- Native Overlay ---"
# Set mount_program = "" in storage.conf
time podman run --rm alpine:latest dd if=/dev/zero of=/tmp/test bs=1M count=100

echo "--- fuse-overlayfs ---"
# Set mount_program = "/usr/bin/fuse-overlayfs" in storage.conf
podman system reset
time podman run --rm alpine:latest dd if=/dev/zero of=/tmp/test bs=1M count=100
```

Native overlay is 2-3x faster than fuse-overlayfs for I/O-heavy workloads.

---

## Troubleshoot Overlay Issues

Common overlay problems and solutions:

```bash
# Error: "overlay is not supported over <filesystem>"
# Solution: Use XFS with ftype=1 or ext4
df -T /var/lib/containers/storage

# Error: "kernel does not support overlay fs"
# Solution: Load the overlay module
sudo modprobe overlay

# Error: "fuse-overlayfs not found"
# Solution: Install fuse-overlayfs
sudo apt-get install -y fuse-overlayfs

# Error: "mounting overlay: permission denied"
# Solution: Check kernel version for rootless support
uname -r  # Need 5.11+ for rootless native overlay

# General troubleshooting: reset and reconfigure
podman system reset
# Edit storage.conf
podman system migrate
podman info | grep -A5 graphDriver
```

Check for common misconfigurations:

```bash
#!/bin/bash
# overlay-check.sh - Verify overlay configuration

echo "=== Overlay Driver Health Check ==="

# Kernel version
KERNEL=$(uname -r)
echo "Kernel: $KERNEL"

# OverlayFS support
if grep -q overlay /proc/filesystems; then
  echo "OverlayFS: Supported"
else
  echo "OverlayFS: NOT supported - load overlay module"
fi

# Storage filesystem
GRAPH=$(podman info --format '{{.Store.GraphRoot}}')
FS_TYPE=$(df -T "$GRAPH" | tail -1 | awk '{print $2}')
echo "Filesystem: $FS_TYPE"

if [ "$FS_TYPE" = "xfs" ]; then
  FTYPE=$(xfs_info "$GRAPH" 2>/dev/null | grep ftype | awk -F= '{print $2}')
  echo "XFS ftype: $FTYPE"
  [ "$FTYPE" != "1" ] && echo "WARNING: XFS ftype must be 1 for overlay"
fi

# Current driver
DRIVER=$(podman info --format '{{.Store.GraphDriverName}}')
echo "Current driver: $DRIVER"

# Metacopy
if [ -f /sys/module/overlay/parameters/metacopy ]; then
  MC=$(cat /sys/module/overlay/parameters/metacopy)
  echo "Metacopy: $MC"
fi

echo "=== Check Complete ==="
```

---

## Conclusion

The overlay storage driver is the optimal choice for Podman container storage. For rootful Podman, it works out of the box on modern kernels with XFS (ftype=1) or ext4 filesystems. For rootless Podman, kernel 5.11+ enables native overlay support with near-rootful performance, while fuse-overlayfs serves as a capable fallback for older kernels. Enable metacopy for faster copy-on-write operations, use the volatile mount option for ephemeral workloads, and minimize layer count in your images. With proper configuration, the overlay driver delivers filesystem performance that is nearly indistinguishable from native host filesystem access.
