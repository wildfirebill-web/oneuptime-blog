# How to Use Overlay Mounts with Podman

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, DevOps, Overlay, Volumes, Filesystem

Description: Learn how to use overlay mounts in Podman to layer directories and provide copy-on-write semantics for container volumes.

---

> Overlay mounts let you layer multiple directories together, allowing containers to see a merged view while keeping the original data untouched through copy-on-write.

Overlay mounts in Podman use the Linux OverlayFS to combine multiple directory layers. This is useful when you want to provide a base set of files that containers can appear to modify without altering the originals.

---

## Understanding Overlay Mounts

An overlay mount combines a lower (read-only) directory with an upper (writable) directory. The container sees a merged view of both. Writes go to the upper layer, leaving the lower layer unchanged.

```bash
# Basic overlay mount syntax with --mount

podman run --rm \
  --mount type=overlay,source=/home/user/base-config,target=/config \
  docker.io/library/alpine:latest ls /config
```

## Creating an Overlay Mount with Multiple Layers

```bash
# Prepare base and override directories
mkdir -p /home/user/base-config
mkdir -p /home/user/override-config

echo "base_setting=true" > /home/user/base-config/app.conf
echo "override_setting=true" > /home/user/override-config/app.conf

# Mount as overlay - the override layer takes precedence
podman run --rm \
  --mount type=overlay,source=/home/user/base-config,target=/config,upperdir=/home/user/override-config \
  docker.io/library/alpine:latest cat /config/app.conf
```

## Using Overlay for Development Workflows

Overlay mounts are useful in development to layer custom configuration over default files without modifying the originals:

```bash
# Base image has default nginx config
# Overlay your custom config on top
podman run -d --name dev-nginx \
  --mount type=overlay,source=/home/user/default-nginx,target=/etc/nginx/conf.d \
  -p 8080:80 \
  docker.io/library/nginx:latest

# Any changes inside the container are written to the upper layer
# The base files remain unchanged
```

## Overlay Volume Options

```bash
# Specify workdir and upperdir explicitly
podman run --rm \
  --mount type=overlay,source=/home/user/lower,target=/data,upperdir=/home/user/upper,workdir=/home/user/work \
  docker.io/library/alpine:latest sh -c "echo hello > /data/newfile.txt && cat /data/newfile.txt"

# The new file appears in the upper directory on the host
cat /home/user/upper/newfile.txt
# Output: hello

# The lower directory is untouched
ls /home/user/lower/newfile.txt
# Output: No such file or directory
```

## Read-Only Overlay Mounts

```bash
# Mount overlay as read-only for strict protection
podman run --rm \
  --mount type=overlay,source=/home/user/base-config,target=/config,readonly \
  docker.io/library/alpine:latest cat /config/app.conf
```

## Overlay vs Bind Mounts

| Feature | Overlay Mount | Bind Mount |
|---------|--------------|------------|
| Copy-on-write | Yes | No |
| Original files protected | Yes | No |
| Multiple layers | Yes | No |
| Performance | Slight overhead | Direct access |

## Combining with SELinux

```bash
# Add SELinux context to overlay mounts on SELinux-enabled systems
podman run --rm \
  -v /home/user/base:/config:O,z \
  docker.io/library/alpine:latest ls /config
# The :O flag can be used as shorthand for overlay type
```

## Summary

Overlay mounts in Podman provide copy-on-write layering for container volumes. They are ideal for scenarios where you need to provide base files that containers can appear to modify without altering the originals. Use explicit `upperdir` and `workdir` options for full control over where modifications are stored.
