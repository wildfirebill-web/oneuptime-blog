# How to Use Overlay Storage Driver with Podman

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, DevOps, Storage, OverlayFS, Performance

Description: Learn how to configure and optimize the overlay storage driver for Podman, including kernel overlay and fuse-overlayfs for rootless containers.

---

> The overlay storage driver is the recommended choice for Podman, providing efficient copy-on-write layer management with minimal disk usage and fast container startup.

OverlayFS is a union filesystem that layers multiple directories on top of each other, making it ideal for container storage. Podman can use either the kernel's native OverlayFS or fuse-overlayfs for environments where kernel support is limited. This guide covers setup, optimization, and troubleshooting of the overlay driver.

---

## Checking Overlay Support

Verify your system supports OverlayFS.

```bash
# Check if the kernel supports overlay
grep -i overlay /proc/filesystems 2>/dev/null
# Expected output: "nodev  overlay"

# Check kernel version (overlay works best with 5.11+)
uname -r

# Check if overlay module is loaded
lsmod | grep overlay 2>/dev/null

# For rootless overlay, check kernel support
cat /proc/sys/kernel/unprivileged_userns_clone 2>/dev/null
# 1 = enabled (required for rootless overlay)

# Check if fuse-overlayfs is available as fallback
which fuse-overlayfs 2>/dev/null && fuse-overlayfs --version
```

## Configuring Kernel Overlay

Set up the native kernel overlay driver.

```bash
# Create storage configuration
mkdir -p ~/.config/containers

cat > ~/.config/containers/storage.conf << 'EOF'
[storage]
# Use the overlay storage driver
driver = "overlay"

[storage.options.overlay]
# Mount options for overlay filesystem
# nodev: Prevent device files in overlay mounts
# metacopy=on: Enable metadata-only copy-up (kernel 4.19+)
mountopt = "nodev,metacopy=on"
EOF

# Reset storage if switching from another driver
# podman system reset --force

# Verify overlay is active
podman info --format '{{.Store.GraphDriverName}}'
```

## Configuring fuse-overlayfs

Use fuse-overlayfs when kernel overlay is not available.

```bash
# Install fuse-overlayfs
# Fedora/RHEL:
# sudo dnf install fuse-overlayfs

# Ubuntu/Debian:
# sudo apt install fuse-overlayfs

# Verify installation
which fuse-overlayfs
fuse-overlayfs --version 2>/dev/null
```

```bash
# Configure fuse-overlayfs in storage.conf
cat > ~/.config/containers/storage.conf << 'EOF'
[storage]
driver = "overlay"

[storage.options.overlay]
# Use fuse-overlayfs for rootless containers
# Required on kernels older than 5.11 for rootless
mount_program = "/usr/bin/fuse-overlayfs"
EOF

# Verify the mount program is being used
podman info --format '{{.Store.GraphDriverName}}'
podman --log-level=debug info 2>&1 | grep -i fuse | head -5
```

## Optimizing Overlay Performance

Tune overlay settings for better performance.

```bash
# Optimized overlay configuration
cat > ~/.config/containers/storage.conf << 'EOF'
[storage]
driver = "overlay"

[storage.options.overlay]
# Enable metacopy for faster layer operations
# Only copies metadata on copy-up, not file contents
mountopt = "nodev,metacopy=on"

# Ignore chown errors in rootless mode
# This avoids performance issues with uid/gid mapping
ignore_chown_errors = "true"

# Force use of native overlay diff
# Avoids falling back to naive diff which is slower
# force_mask = "0755"
EOF
```

```bash
# Test performance with the optimized configuration
echo "Driver: $(podman info --format '{{.Store.GraphDriverName}}')"

# Time an image pull to measure layer handling speed
time podman pull nginx:alpine 2>/dev/null

# Time container creation and startup
time podman run --rm alpine echo "Performance test"

# Check disk efficiency - overlay shares layers
podman pull alpine:3.18 2>/dev/null
podman pull alpine:3.19 2>/dev/null
podman system df -v 2>/dev/null | head -15
```

## Understanding Overlay Layers

See how overlay manages container filesystem layers.

```bash
# Pull an image and inspect its layers
podman pull alpine:latest
podman image inspect alpine:latest --format '{{.RootFS.Layers}}'

# View the overlay storage structure
GRAPH_ROOT=$(podman info --format '{{.Store.GraphRoot}}')
echo "Storage root: $GRAPH_ROOT"

# List overlay layers
ls "$GRAPH_ROOT/overlay/" 2>/dev/null | head -10

# Each layer has three directories:
# diff/   - Layer contents (the actual files)
# merged/ - Merged view (when container is running)
# work/   - Overlay working directory
ls "$GRAPH_ROOT/overlay/" 2>/dev/null | head -1 | xargs -I{} ls "$GRAPH_ROOT/overlay/{}" 2>/dev/null
```

## Rootless Overlay Configuration

Configure overlay specifically for rootless Podman.

```bash
# Complete rootless overlay configuration
cat > ~/.config/containers/storage.conf << 'EOF'
[storage]
driver = "overlay"
graphroot = "$HOME/.local/share/containers/storage"
runroot = "$XDG_RUNTIME_DIR/containers"

[storage.options]
# Enable automatic user namespace remapping
# This is essential for rootless overlay
pull_options = {enable_partial_images = "true", use_hard_links = "false", ostree_repos=""}

[storage.options.overlay]
# Use kernel overlay if supported, otherwise fuse-overlayfs
# mount_program = "/usr/bin/fuse-overlayfs"

# Mount options
mountopt = "nodev,metacopy=on"

# Ignore chown errors for rootless
ignore_chown_errors = "true"
EOF

# Test rootless operation
podman info --format '{{.Host.Security.Rootless}}'
podman run --rm alpine id
```

## Troubleshooting Overlay Issues

Fix common overlay driver problems.

```bash
# Issue: "overlay is not supported over filesystem"
# Check the backing filesystem
df -T "$(podman info --format '{{.Store.GraphRoot}}')"
# overlay works on: ext4, xfs, btrfs, tmpfs
# overlay does NOT work on: nfs, ecryptfs, aufs

# Issue: "kernel does not support overlay fs"
# Solution: Use fuse-overlayfs
# Ensure fuse-overlayfs is installed and configured

# Issue: "permission denied" on overlay mount
# Check user namespaces are enabled
sysctl -n kernel.unprivileged_userns_clone 2>/dev/null

# General debugging
podman --log-level=debug run --rm alpine echo "debug" 2>&1 | grep -i "overlay\|storage\|mount" | head -10

# Fix corrupted overlay storage
podman system reset --force
podman info --format '{{.Store.GraphDriverName}}'
podman run --rm alpine echo "Overlay recovered"
```

## Summary

The overlay storage driver is Podman's recommended choice, providing efficient copy-on-write layer management. Use native kernel overlay on modern kernels (5.11+) with `metacopy=on` for best performance, or fall back to fuse-overlayfs on older systems. Enable `ignore_chown_errors` for rootless operation, and ensure your backing filesystem (ext4 or xfs) supports overlay. Always verify the configuration with `podman info` and test with a container run after making changes.
