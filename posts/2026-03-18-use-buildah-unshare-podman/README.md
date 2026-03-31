# How to Use Buildah Unshare with Podman

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Container, DevOps, Buildah, unshare, Rootless, User Namespaces, Security

Description: Learn how to use buildah unshare to run privileged container build operations in rootless mode using user namespaces.

---

> Buildah unshare lets you perform root-level operations inside containers without actual root privileges on the host.

Running containers and building images as a non-root user is a core security feature of Podman and Buildah. However, some build operations require root-like access to the container filesystem, such as mounting filesystems or changing file ownership. The `buildah unshare` command creates a user namespace that maps your non-root user to root inside the namespace, enabling these operations without actual root access. This guide explains how unshare works and when to use it.

---

## Understanding User Namespaces

```bash
# Check your current user identity

id

# Check your subordinate UID/GID mappings
# These define the range of UIDs available for user namespaces
cat /etc/subuid | grep $(whoami)
cat /etc/subgid | grep $(whoami)

# Typical output: username:100000:65536
# This means UIDs 100000-165535 are available for your user namespaces

# If these files are empty or missing, set them up:
# sudo usermod --add-subuids 100000-165535 --add-subgids 100000-165535 $(whoami)
```

## Basic Unshare Usage

### Enter an Unshare Session

```bash
# Start an interactive unshare session
# Inside this session, you appear as root but have no actual root privileges
buildah unshare

# Inside the unshare session:
id
# Output: uid=0(root) gid=0(root) groups=0(root)
# You appear as root, but this is only within the user namespace

whoami
# Output: root

# Exit the unshare session
exit
```

### Run a Command in Unshare

```bash
# Run a single command inside unshare without entering interactively
buildah unshare id
# Output: uid=0(root) gid=0(root)

# Run a script inside unshare
buildah unshare bash -c 'echo "I am $(whoami) with UID $(id -u)"'

# Check the namespace mapping
buildah unshare cat /proc/self/uid_map
```

## Using Unshare with Buildah Mount

The most common use case for `buildah unshare` is enabling `buildah mount` in rootless mode.

```bash
# Without unshare, buildah mount fails in rootless mode
container=$(buildah from alpine:3.19)

# This will fail for non-root users:
# buildah mount $container
# Error: mount is not supported in rootless mode

# Instead, use buildah unshare to access the mount
buildah unshare bash -c "
  # Create a container inside the unshare session
  ctr=\$(buildah from alpine:3.19)

  # Mount the container filesystem
  mnt=\$(buildah mount \$ctr)
  echo \"Mounted at: \$mnt\"

  # Now you can interact with the filesystem
  ls \$mnt/
  cat \$mnt/etc/alpine-release

  # Create files with proper ownership
  echo 'built in unshare' > \$mnt/build-info.txt
  chown 1000:1000 \$mnt/build-info.txt
  ls -la \$mnt/build-info.txt

  # Unmount and commit
  buildah umount \$ctr
  buildah commit \$ctr unshare-demo:latest
  buildah rm \$ctr
"

# The image is available outside the unshare session
podman images unshare-demo --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"
```

## File Ownership in Unshare

```bash
# Unshare enables proper file ownership management
buildah unshare bash -c '
  ctr=$(buildah from ubuntu:22.04)
  mnt=$(buildah mount $ctr)

  # Create a user in the container
  echo "appuser:x:1000:1000:App User:/home/appuser:/bin/bash" >> $mnt/etc/passwd
  echo "appuser:x:1000:" >> $mnt/etc/group

  # Create application directories with proper ownership
  mkdir -p $mnt/app/{data,logs,config}

  # Set ownership (this requires root in the namespace)
  chown -R 1000:1000 $mnt/app/

  # Create files owned by the app user
  echo "app data" > $mnt/app/data/initial.dat
  chown 1000:1000 $mnt/app/data/initial.dat

  # Set restrictive permissions
  chmod 750 $mnt/app/
  chmod 640 $mnt/app/data/initial.dat

  # Verify ownership and permissions
  ls -la $mnt/app/
  ls -la $mnt/app/data/

  buildah umount $ctr
  buildah config --user appuser $ctr
  buildah commit $ctr owned-app:latest
  buildah rm $ctr
'

# Verify the ownership is preserved when running
podman run --rm owned-app:latest ls -la /app/
podman run --rm owned-app:latest ls -la /app/data/
```

## Scripted Builds with Unshare

```bash
cat << 'SCRIPT' > /tmp/unshare-build.sh
#!/bin/bash
# Complete build script using buildah unshare for rootless builds
set -euo pipefail

IMAGE_NAME="${1:-myservice}"
IMAGE_TAG="${2:-latest}"

buildah unshare bash -c "
  set -euo pipefail

  echo '=== Starting rootless build ==='

  # Create the working container
  ctr=\$(buildah from python:3.12-slim)
  mnt=\$(buildah mount \$ctr)

  # Create application structure with proper ownership
  mkdir -p \$mnt/app/{src,config,static}
  mkdir -p \$mnt/var/log/app

  # Create a non-root user
  echo 'appuser:x:1000:1000::/app:/sbin/nologin' >> \$mnt/etc/passwd
  echo 'appgroup:x:1000:' >> \$mnt/etc/group

  # Write application files directly to the filesystem
  cat << 'PYEOF' > \$mnt/app/src/main.py
from http.server import HTTPServer, SimpleHTTPRequestHandler
import os
print(f'Running as UID: {os.getuid()}')
HTTPServer(('0.0.0.0', 8080), SimpleHTTPRequestHandler).serve_forever()
PYEOF

  # Set ownership for all application files
  chown -R 1000:1000 \$mnt/app/
  chown -R 1000:1000 \$mnt/var/log/app/
  chmod -R 755 \$mnt/app/src/
  chmod -R 700 \$mnt/var/log/app/

  # Unmount before configuring
  buildah umount \$ctr

  # Configure the image
  buildah config --workingdir /app/src \$ctr
  buildah config --user 1000:1000 \$ctr
  buildah config --port 8080 \$ctr
  buildah config --entrypoint '[\"python3\", \"main.py\"]' \$ctr
  buildah config --label build-type=rootless \$ctr

  # Commit the image
  buildah commit --squash \$ctr ${IMAGE_NAME}:${IMAGE_TAG}
  buildah rm \$ctr

  echo '=== Build complete ==='
"

echo "Image built: ${IMAGE_NAME}:${IMAGE_TAG}"
podman images "${IMAGE_NAME}" --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"
SCRIPT
chmod +x /tmp/unshare-build.sh

# Run the build script
/tmp/unshare-build.sh myservice v1.0
```

## Debugging with Unshare

```bash
# Use unshare to debug filesystem issues in containers
buildah unshare bash -c '
  ctr=$(buildah from alpine:3.19)
  mnt=$(buildah mount $ctr)

  echo "=== Filesystem permissions audit ==="
  echo "World-writable files:"
  find $mnt -perm -o+w -type f 2>/dev/null | head -5

  echo ""
  echo "SUID binaries:"
  find $mnt -perm -4000 -type f 2>/dev/null

  echo ""
  echo "Files owned by root:"
  find $mnt/app -user 0 2>/dev/null | head -5

  echo ""
  echo "Disk usage by directory:"
  du -sh $mnt/* 2>/dev/null | sort -rh | head -10

  buildah umount $ctr
  buildah rm $ctr
'
```

## Unshare Environment Variables

```bash
# Pass environment variables into the unshare session
export BUILD_VERSION="3.0.0"
export BUILD_ENV="staging"

buildah unshare bash -c '
  echo "Building version: $BUILD_VERSION for $BUILD_ENV"
  ctr=$(buildah from alpine:3.19)
  buildah config --label version="$BUILD_VERSION" $ctr
  buildah config --label environment="$BUILD_ENV" $ctr
  buildah commit $ctr versioned-app:$BUILD_VERSION
  buildah rm $ctr
'

podman inspect versioned-app:$BUILD_VERSION --format '{{.Config.Labels}}'
```

## Cleaning Up

```bash
buildah rm --all 2>/dev/null
podman rmi unshare-demo:latest owned-app:latest myservice:v1.0 versioned-app:3.0.0 2>/dev/null
rm -f /tmp/unshare-build.sh
```

## Summary

The `buildah unshare` command is essential for rootless container builds that require filesystem-level operations like mounting, changing ownership, or setting special permissions. It creates a user namespace where your non-root user is mapped to root, providing the necessary capabilities without actual root privileges on the host. This maintains the security benefits of rootless containers while enabling advanced build workflows. Combining unshare with buildah mount gives you direct filesystem access for inspection, modification, and debugging of container images.
