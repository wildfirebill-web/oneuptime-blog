# How to Use VFS Storage Driver with Podman

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, DevOps, Storage, VFS, Compatibility

Description: Learn how to configure and use the VFS storage driver with Podman for maximum compatibility on systems where overlay is not available.

---

> The VFS storage driver provides maximum compatibility by using simple directory copies instead of filesystem-level layering, working on any Linux filesystem.

While the overlay driver is recommended for most environments, the VFS (Virtual File System) driver serves as a universal fallback. VFS works by creating a complete copy of each layer, making it slower and more disk-intensive but compatible with every filesystem. This guide explains when to use VFS and how to configure it properly.

---

## When to Use VFS

VFS is the right choice in specific scenarios.

```bash
# Check if your current driver is already VFS
podman info --format '{{.Store.GraphDriverName}}'

# Scenarios where VFS is appropriate:
# 1. Network filesystems (NFS, CIFS) that do not support overlay
# 2. Encrypted filesystems (ecryptfs) incompatible with overlay
# 3. Nested container environments (containers inside containers)
# 4. Testing and debugging storage issues
# 5. Systems with very old kernels (pre-3.18)

# Check your filesystem type
df -T "$(podman info --format '{{.Store.GraphRoot}}')"
```

## Configuring VFS Driver

Set up VFS as the storage driver.

```bash
# Create storage configuration
mkdir -p ~/.config/containers

cat > ~/.config/containers/storage.conf << 'EOF'
[storage]
# Use VFS storage driver
# VFS makes complete copies of each layer
# Compatible with all filesystems
driver = "vfs"

[storage.options.vfs]
# VFS has minimal configuration options
# No special filesystem support required
EOF

# Reset storage when switching from another driver
podman system reset --force

# Verify VFS is active
podman info --format '{{.Store.GraphDriverName}}'
```

## Understanding VFS Behavior

VFS creates full copies of each filesystem layer.

```bash
# Pull an image and observe VFS storage behavior
podman pull alpine:latest

# Check disk usage - VFS uses more space than overlay
podman system df

# VFS directory structure
GRAPH_ROOT=$(podman info --format '{{.Store.GraphRoot}}')
echo "Graph root: $GRAPH_ROOT"
ls "$GRAPH_ROOT/vfs/" 2>/dev/null | head -10

# Each layer is a complete directory copy
# Unlike overlay, there is no shared layer mechanism
du -sh "$GRAPH_ROOT/vfs/" 2>/dev/null
```

```bash
# Compare disk usage: pull two related images
podman pull alpine:3.18 2>/dev/null
podman pull alpine:3.19 2>/dev/null

# With VFS, shared layers are duplicated
podman system df -v 2>/dev/null | head -15

# Check total storage used
du -sh "$GRAPH_ROOT" 2>/dev/null
```

## VFS on Network Filesystems

Configure VFS for NFS or CIFS mounted storage.

```bash
# Example: Configure VFS for NFS-backed storage
cat > ~/.config/containers/storage.conf << 'EOF'
[storage]
driver = "vfs"

# Point storage to an NFS mount
# Ensure the NFS mount is reliable and has low latency
graphroot = "/nfs/containers/storage"
runroot = "$XDG_RUNTIME_DIR/containers"

[storage.options.vfs]
# No special options needed
EOF

# Note: Keep runroot on a local filesystem for performance
# Runtime data needs fast access and is ephemeral

# Reset and verify
podman system reset --force
podman info --format '{{.Store.GraphDriverName}}'
podman info --format '{{.Store.GraphRoot}}'
```

## VFS in Nested Container Environments

Use VFS when running Podman inside containers.

```bash
# VFS is often required for Docker-in-Docker or Podman-in-Podman
# The inner container engine cannot use overlay on an overlay filesystem

# Example: Running Podman inside a Podman container
podman run --rm -it \
    --security-opt label=disable \
    --device /dev/fuse \
    quay.io/podman/stable \
    podman info --format '{{.Store.GraphDriverName}}'

# If the inner Podman uses VFS, configure it explicitly
cat > /tmp/inner-storage.conf << 'EOF'
[storage]
driver = "vfs"
EOF

# Mount the config into the inner container
podman run --rm \
    --security-opt label=disable \
    -v /tmp/inner-storage.conf:/etc/containers/storage.conf:ro \
    quay.io/podman/stable \
    podman info --format '{{.Store.GraphDriverName}}'
```

## Managing Disk Space with VFS

VFS uses significantly more disk space, so active management is essential.

```bash
# Monitor disk usage regularly
podman system df

# Set up regular cleanup
# Remove unused images
podman image prune -a -f

# Remove stopped containers
podman container prune -f

# Remove unused volumes
podman volume prune -f

# Full system cleanup
podman system prune -a -f

# Check disk usage after cleanup
podman system df
du -sh "$(podman info --format '{{.Store.GraphRoot}}')" 2>/dev/null
```

```bash
# Automate cleanup with a cron job or systemd timer
# Example cron entry (add with crontab -e):
# 0 3 * * * podman system prune -a -f > /dev/null 2>&1

# Monitor storage growth over time
echo "Current VFS storage usage:"
du -sh "$(podman info --format '{{.Store.GraphRoot}}')" 2>/dev/null
echo "Number of images: $(podman images -q | wc -l)"
echo "Number of containers: $(podman ps -aq | wc -l)"
```

## Performance Considerations

Understand and mitigate VFS performance limitations.

```bash
# VFS performance characteristics:
# - Slower image pulls (full layer copies)
# - More disk I/O during container creation
# - Higher disk usage (no layer sharing)
# - Slower container startup

# Benchmark container startup with VFS
time podman run --rm alpine echo "VFS startup test"

# Benchmark image pull
time podman pull nginx:alpine 2>/dev/null

# Mitigation strategies:
# 1. Use fast storage (SSD/NVMe)
# 2. Minimize the number of cached images
# 3. Use multi-stage builds to reduce image size
# 4. Clean up regularly with podman system prune
```

## Switching Away from VFS

Migrate to a more efficient driver when possible.

```bash
# Save important images before switching
podman save -o /tmp/my-images.tar $(podman images -q) 2>/dev/null

# Switch to overlay driver
cat > ~/.config/containers/storage.conf << 'EOF'
[storage]
driver = "overlay"

[storage.options.overlay]
mountopt = "nodev,metacopy=on"
EOF

# Reset storage
podman system reset --force

# Restore images
podman load -i /tmp/my-images.tar 2>/dev/null

# Verify the switch
podman info --format '{{.Store.GraphDriverName}}'

# Compare disk usage after switching to overlay
podman system df
```

## Summary

The VFS storage driver is Podman's universal fallback, working on any filesystem including NFS, ecryptfs, and nested container environments. It trades performance and disk efficiency for maximum compatibility by creating full copies of each layer. Use VFS when overlay is not available, but manage disk space aggressively with regular `podman system prune` operations. When your environment supports overlay, switch to it for significantly better performance and disk usage.
