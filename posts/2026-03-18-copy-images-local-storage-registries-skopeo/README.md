# How to Copy Images Between Local Storage and Registries with Skopeo

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, DevOps, Skopeo, Registry, Local Storage, Image Transfer

Description: Learn how to use Skopeo to move container images between Podman local storage, local directories, and remote registries using different transport types.

---

> Skopeo supports multiple transport types, letting you move images between Podman's local storage, filesystem directories, OCI layouts, and remote registries with a single command.

Container images do not always live in remote registries. Sometimes you need to export them to a directory for offline transfer, push a locally built image to a registry, or import an OCI archive from a colleague. Skopeo handles all of these scenarios through its transport system. This guide covers how to use Skopeo to move images between Podman local storage, directories, OCI layouts, and registries.

---

## Understanding Skopeo Transports

Skopeo uses transport prefixes to identify where images live.

```bash
# Transport types:
# docker://         — Remote Docker-compatible registry
# containers-storage: — Local Podman/CRI-O storage
# dir:              — Local directory with raw blobs
# oci:              — OCI image layout directory
# docker-archive:   — Docker tar archive file
# oci-archive:      — OCI tar archive file
```

## Copying from Local Podman Storage to a Registry

After building an image with Podman, push it to a registry using Skopeo.

```bash
# Build an image locally with Podman
podman build -t myapp:v1.0 .

# Copy from Podman's local storage to a remote registry
skopeo copy \
  containers-storage:localhost/myapp:v1.0 \
  docker://registry.example.com/myapp:v1.0

# Verify the image is in the registry
skopeo inspect docker://registry.example.com/myapp:v1.0 | jq '.Digest'
```

## Copying from a Registry to Local Podman Storage

Pull an image from a registry directly into Podman storage without using `podman pull`.

```bash
# Copy from a registry to local Podman storage
skopeo copy \
  docker://docker.io/library/nginx:1.25 \
  containers-storage:nginx:1.25

# Verify the image is in Podman's local storage
podman images | grep nginx
```

## Exporting Images to a Directory

The `dir:` transport saves image layers and manifest to a flat directory structure.

```bash
# Export an image from a registry to a local directory
mkdir -p /tmp/nginx-export
skopeo copy \
  docker://docker.io/library/nginx:1.25 \
  dir:/tmp/nginx-export

# Examine the exported contents
ls -la /tmp/nginx-export/
# You will see manifest.json, version, and layer blob files
```

## Importing Images from a Directory

Copy an image from a local directory back to Podman storage or a registry.

```bash
# Import from a directory into Podman local storage
skopeo copy \
  dir:/tmp/nginx-export \
  containers-storage:nginx-imported:1.25

# Import from a directory to a remote registry
skopeo copy \
  dir:/tmp/nginx-export \
  docker://registry.example.com/nginx:1.25
```

## Working with OCI Layouts

OCI layout is a standard format for storing images on the filesystem.

```bash
# Export an image to OCI layout format
skopeo copy \
  docker://docker.io/library/alpine:3.19 \
  oci:/tmp/alpine-oci:latest

# View the OCI layout structure
ls /tmp/alpine-oci/
# Expect: blobs/ index.json oci-layout

# Import from OCI layout to Podman storage
skopeo copy \
  oci:/tmp/alpine-oci:latest \
  containers-storage:alpine-from-oci:3.19

# Import from OCI layout to a registry
skopeo copy \
  oci:/tmp/alpine-oci:latest \
  docker://registry.example.com/alpine:3.19
```

## Working with Archive Files

Archive transports create single tar files that are easy to transfer.

```bash
# Create a Docker-format tar archive
skopeo copy \
  docker://docker.io/library/redis:7 \
  docker-archive:/tmp/redis-7.tar:redis:7

# Create an OCI-format tar archive
skopeo copy \
  docker://docker.io/library/redis:7 \
  oci-archive:/tmp/redis-7-oci.tar:redis:7

# Import a Docker archive into Podman storage
skopeo copy \
  docker-archive:/tmp/redis-7.tar \
  containers-storage:redis:7

# Import an OCI archive to a registry
skopeo copy \
  oci-archive:/tmp/redis-7-oci.tar \
  docker://registry.example.com/redis:7
```

## Offline Image Transfer Workflow

Use Skopeo for transferring images to air-gapped environments.

```bash
#!/bin/bash
# export-for-airgap.sh — Export images for offline transfer

EXPORT_DIR="/media/usb-drive/images"
mkdir -p "$EXPORT_DIR"

# List of images to export
IMAGES=(
  "docker.io/library/nginx:1.25"
  "docker.io/library/postgres:16"
  "docker.io/library/redis:7"
)

# Export each image as an OCI archive
for IMG in "${IMAGES[@]}"; do
  # Create a safe filename from the image reference
  FILENAME=$(echo "$IMG" | tr '/:' '_')
  echo "Exporting ${IMG} to ${FILENAME}.tar"
  skopeo copy \
    "docker://${IMG}" \
    "oci-archive:${EXPORT_DIR}/${FILENAME}.tar"
done

echo "Export complete. Transfer ${EXPORT_DIR} to the air-gapped system."
```

```bash
#!/bin/bash
# import-airgap.sh — Import images on the air-gapped system

IMPORT_DIR="/media/usb-drive/images"
REGISTRY="airgapped-registry.internal:5000"

# Import each archive into the local registry
for ARCHIVE in "${IMPORT_DIR}"/*.tar; do
  echo "Importing ${ARCHIVE}..."
  skopeo copy \
    --dest-tls-verify=false \
    "oci-archive:${ARCHIVE}" \
    "docker://${REGISTRY}/$(basename "${ARCHIVE}" .tar)"
done

echo "Import complete."
```

## Summary

Skopeo's transport system provides complete flexibility for moving container images between different storage types. You can copy images between Podman local storage, directories, OCI layouts, tar archives, and remote registries. This makes it invaluable for air-gapped transfers, backup workflows, and migrating images between environments. Combined with Podman, Skopeo gives you a complete toolkit for managing container images across any storage medium.
