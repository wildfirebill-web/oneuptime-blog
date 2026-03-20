# How to Browse Volume Contents in Portainer (Swarm/Agent) - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Volumes, Storage, DevOps

Description: Learn how to browse and view the contents of Docker volumes directly in Portainer using the volume browser feature available with Swarm and Agent deployments.

## Introduction

Portainer Business Edition and environments using the Portainer Agent provide a built-in volume browser that lets you navigate volume contents, view files, and download or upload files - all from the web interface. This is extremely useful for debugging, data verification, and quick file management without SSH access.

## Prerequisites

- Portainer with Portainer Agent deployed on the target host
- Or: Portainer in Docker Swarm mode
- Named volume to browse

## Step 1: Access the Volume Browser

The volume browser is available in environments managed via the Portainer Agent:

1. Navigate to the Docker environment managed by the Agent.
2. Go to **Volumes** in the sidebar.
3. Find the volume you want to browse.
4. Click the volume name.
5. Click the **Browse** button (folder icon).

## Step 2: Navigate the Volume

The volume browser shows:
- File and directory listings
- File sizes
- Last modified dates
- File/directory icons

Navigate by:
- Clicking directories to enter them
- Using the breadcrumb navigation at the top
- Using the back button

## Step 3: View File Contents

1. In the volume browser, click on a file.
2. Portainer displays the file content in a viewer.
3. For text files (configs, logs), you can view the content directly.
4. Binary files may not display correctly.

## Step 4: Download Files from a Volume

To download a specific file:

1. Navigate to the file in the volume browser.
2. Click the **Download** button or right-click the file.
3. The file downloads to your local machine.

This is useful for:
- Downloading config files for review.
- Exporting database dumps stored in volumes.
- Retrieving log files.

## Step 5: Upload Files to a Volume

To add or update files in a volume:

1. Navigate to the target directory in the browser.
2. Click **Upload** (or drag-and-drop).
3. Select the file from your local machine.
4. The file is uploaded to the volume.

Use cases:
- Uploading updated configuration files.
- Adding SSL certificates to a volume.
- Uploading seed data files.

## Alternative: Browse via Container Console

If the volume browser isn't available, use a running container:

```bash
# If you have a container with the volume mounted, use exec:

docker exec -it my-container ls -la /app/data/

# Or open the Portainer console:
# 1. Navigate to Containers > (container with the volume) > Console
# 2. Navigate to the volume mount point
ls -la /app/data/
cat /app/data/config.yaml
find /app/data -name "*.log" | head -10
```

## Alternative: Temporary Container for Volume Browsing

For any environment (not just Agent-managed), use a temporary browser container:

```bash
# Create a temporary container to browse volume contents:
docker run --rm -it \
  -v my-volume:/data \
  alpine:latest \
  /bin/sh

# Now inside the container:
ls -la /data
cat /data/config/app.yaml
du -sh /data/*
```

Or for non-interactive browsing:

```bash
# List all files in the volume:
docker run --rm \
  -v my-volume:/data \
  alpine:latest \
  find /data -type f | sort

# Get volume size:
docker run --rm \
  -v my-volume:/data \
  alpine:latest \
  du -sh /data

# View a specific file:
docker run --rm \
  -v my-volume:/data \
  alpine:latest \
  cat /data/config/app.yaml

# Copy a file out of the volume:
docker run --rm \
  -v my-volume:/data \
  -v $(pwd):/output \
  alpine:latest \
  cp /data/important.db /output/
```

## Volume File Management Script

For frequent volume operations without a browser:

```bash
#!/bin/bash
# volume-browser.sh
# Simple CLI volume browser

VOLUME="${1:?Volume name required}"
ACTION="${2:-list}"
PATH_IN_VOLUME="${3:-/}"

case "$ACTION" in
    list)
        docker run --rm -v "${VOLUME}:/vol" alpine ls -la "/vol${PATH_IN_VOLUME}"
        ;;
    cat)
        docker run --rm -v "${VOLUME}:/vol" alpine cat "/vol${PATH_IN_VOLUME}"
        ;;
    tree)
        docker run --rm -v "${VOLUME}:/vol" alpine find "/vol${PATH_IN_VOLUME}" | sort
        ;;
    size)
        docker run --rm -v "${VOLUME}:/vol" alpine du -sh "/vol${PATH_IN_VOLUME}"
        ;;
    copy-out)
        local dest="${4:-$(basename ${PATH_IN_VOLUME})}"
        docker run --rm \
          -v "${VOLUME}:/vol" \
          -v "$(pwd):/output" \
          alpine cp "/vol${PATH_IN_VOLUME}" "/output/${dest}"
        echo "Copied to: ${dest}"
        ;;
    copy-in)
        local src="${4:?Source file required}"
        docker run --rm \
          -v "${VOLUME}:/vol" \
          -v "$(dirname ${src}):/input" \
          alpine cp "/input/$(basename ${src})" "/vol${PATH_IN_VOLUME}"
        echo "Copied to volume: ${PATH_IN_VOLUME}"
        ;;
    *)
        echo "Usage: $0 <volume> <list|cat|tree|size|copy-out|copy-in> [path] [extra]"
        ;;
esac
```

Usage:

```bash
./volume-browser.sh my-volume list /config/
./volume-browser.sh my-volume cat /config/app.yaml
./volume-browser.sh my-volume size /
./volume-browser.sh my-volume copy-out /config/app.yaml ./local-backup.yaml
```

## Step 6: Access Volume Data Directly on Host

For direct access (when you have host access):

```bash
# Find volume path:
docker volume inspect my-volume | jq '.[].Mountpoint'
# /var/lib/docker/volumes/my-volume/_data

# Browse directly (requires root or docker group):
sudo ls -la /var/lib/docker/volumes/my-volume/_data/

# View a file:
sudo cat /var/lib/docker/volumes/my-volume/_data/config.yaml
```

## Conclusion

Portainer's volume browser (available with Agent-based environments) provides a convenient web-based way to inspect volume contents without SSH or CLI access. For non-Agent environments, use temporary Alpine containers to browse and manage volume contents. Both approaches give you full visibility into what's stored in your Docker volumes for debugging, data verification, and file management.
