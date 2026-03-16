# How to Import a Filesystem as an Image with Podman

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, DevOps, Container Images, Import, Filesystem

Description: Learn how to import a filesystem tarball as a container image using Podman, enabling custom base images and container migration workflows.

---

> Importing filesystems as images lets you create container images from any root filesystem without a Containerfile.

Sometimes you need to create a container image from a raw filesystem rather than building from a Containerfile. This is common when migrating legacy systems, creating minimal base images, or converting virtual machine snapshots into containers. Podman's `podman import` command takes a filesystem tarball and turns it into a usable container image. This guide covers the full workflow.

---

## Understanding podman import vs podman load

These two commands serve different purposes and are often confused.

- `podman load` restores a previously saved image with all its layers and metadata intact.
- `podman import` creates a new single-layer image from a raw filesystem tarball.

```bash
# podman load: restores a saved image (preserves layers)
podman load -i saved-image.tar

# podman import: creates a new image from a filesystem export
podman import filesystem.tar mynewimage:latest
```

## Creating a Filesystem Tarball

Before you can import, you need a tarball of a filesystem. There are several ways to create one.

```bash
# Method 1: Export from an existing container
podman create --name temp-container docker.io/library/alpine:latest
podman export temp-container -o alpine-filesystem.tar
podman rm temp-container

# Method 2: Create a tarball from a directory
# (e.g., a chroot environment or debootstrap output)
sudo tar -C /path/to/rootfs -cf custom-rootfs.tar .

# Method 3: From a running system's filesystem
sudo tar --exclude=/proc --exclude=/sys --exclude=/dev \
  --exclude=/run --exclude=/tmp \
  -cf system-export.tar -C / .
```

## Basic Import

The simplest import creates a new image from a tarball.

```bash
# Import a filesystem tarball as a new image
podman import alpine-filesystem.tar myalpine:latest

# Verify the image was created
podman images myalpine
# REPOSITORY   TAG     IMAGE ID      CREATED        SIZE
# localhost/myalpine  latest  abc123def456  5 seconds ago  7.8 MB
```

## Importing from Standard Input

You can pipe a tarball directly into `podman import`.

```bash
# Import from a compressed tarball
gunzip -c filesystem.tar.gz | podman import - myimage:latest

# Import from a remote URL
curl -sSL https://example.com/rootfs.tar | podman import - myimage:v1.0
```

## Importing from a URL

Podman can import directly from a URL without downloading the file first.

```bash
# Import directly from a URL
podman import https://example.com/rootfs.tar.gz myimage:latest
```

## Setting Image Configuration During Import

You can set container configuration at import time using the `--change` or `-c` flag. This embeds runtime defaults into the image.

```bash
# Set the default command
podman import -c "CMD /bin/sh" filesystem.tar myimage:latest

# Set an entrypoint
podman import -c "ENTRYPOINT /usr/local/bin/app" filesystem.tar myapp:latest

# Set exposed ports
podman import -c "EXPOSE 8080" filesystem.tar mywebapp:latest

# Set environment variables
podman import -c "ENV APP_ENV=production" filesystem.tar myapp:latest

# Set the working directory
podman import -c "WORKDIR /app" filesystem.tar myapp:latest

# Combine multiple changes
podman import \
  -c "CMD /usr/sbin/nginx -g 'daemon off;'" \
  -c "EXPOSE 80" \
  -c "ENV NGINX_VERSION=1.27" \
  -c "WORKDIR /etc/nginx" \
  filesystem.tar mynginx:custom
```

## Adding a Commit Message

You can attach a message to the imported image for documentation purposes.

```bash
# Import with a descriptive message
podman import \
  --message "Imported from production server backup 2026-03-16" \
  filesystem.tar myserver:backup

# View the message in the image history
podman history myserver:backup
```

## Creating Minimal Base Images

One of the most practical uses of `podman import` is creating minimal base images.

```bash
# Create a minimal Alpine-based filesystem
mkdir -p /tmp/minimal-rootfs
# Use an Alpine container to install just what you need
podman run --rm -v /tmp/minimal-rootfs:/output docker.io/library/alpine:latest sh -c "
  apk add --no-cache --root /output --initdb busybox
"

# Create a tarball from the minimal filesystem
tar -C /tmp/minimal-rootfs -cf minimal.tar .

# Import as a base image
podman import -c "CMD /bin/sh" minimal.tar minimal-base:latest

# Test the minimal image
podman run --rm minimal-base:latest echo "Hello from minimal image"
```

## Importing a Debootstrap Filesystem

For Debian-based minimal images, you can use debootstrap.

```bash
# Create a minimal Debian filesystem
sudo debootstrap --variant=minbase bookworm /tmp/debian-rootfs

# Create a tarball
sudo tar -C /tmp/debian-rootfs -cf debian-minimal.tar .

# Import into Podman
podman import \
  -c "CMD /bin/bash" \
  -c "ENV DEBIAN_FRONTEND=noninteractive" \
  debian-minimal.tar debian-minimal:bookworm

# Test it
podman run --rm debian-minimal:bookworm cat /etc/debian_version
```

## Migrating from Docker Export to Podman

If you have containers exported from Docker, you can import them into Podman.

```bash
# On the Docker host: export a container
docker export my-running-container -o container-export.tar

# Transfer the file to the Podman host
scp container-export.tar user@podman-host:/tmp/

# On the Podman host: import the filesystem
podman import \
  -c "CMD /app/start.sh" \
  -c "EXPOSE 3000" \
  /tmp/container-export.tar migrated-app:latest

# Run the migrated container
podman run -d -p 3000:3000 migrated-app:latest
```

## Verifying the Imported Image

After import, inspect the image to confirm its configuration.

```bash
# Check image details
podman inspect myimage:latest

# Check the image history (will show a single layer)
podman history myimage:latest

# Run a quick test
podman run --rm myimage:latest cat /etc/os-release
```

## Summary

The `podman import` command bridges the gap between raw filesystems and container images. It is essential for creating custom base images, migrating legacy systems into containers, and working with filesystem exports. Combined with the `--change` flag, you can set all necessary container defaults at import time, making the resulting image ready to run immediately.
