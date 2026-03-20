# How to Fix "Unable to Retrieve Image Details" After Docker Update

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Troubleshooting, Images, Docker Update

Description: Resolve "Unable to Retrieve Image Details" errors in Portainer that appear after Docker Engine updates, caused by API changes, image format differences, and metadata compatibility issues.

## Introduction

After updating Docker Engine, Portainer may show "Unable to Retrieve Image Details" for some or all images. This is typically caused by Docker API version changes that affect how image metadata is formatted and returned. This guide explains the fixes.

## Step 1: Check the Specific Error in Logs

```bash
# Check Portainer logs for image-related errors
docker logs portainer 2>&1 | grep -i "image\|retrieve\|inspect\|error" | tail -30

# Check Docker daemon logs for relevant errors
journalctl -u docker --since "30 minutes ago" | grep -i "image\|manifest\|digest"
```

## Step 2: Test Image Inspection from CLI

```bash
# If CLI works but Portainer doesn't, it's a Portainer version issue
docker image inspect nginx:latest

# Test the Docker API directly
curl --unix-socket /var/run/docker.sock \
  http://localhost/v1.45/images/nginx:latest/json | jq '.'

# Check if the API returns valid data
# If this fails, it's a Docker API issue
```

## Step 3: Update Portainer to Latest Version

```bash
# Check current Portainer version
docker exec portainer /app/portainer --version

# Pull the latest version
docker pull portainer/portainer-ce:latest

# Update (data volume is preserved)
docker stop portainer && docker rm portainer
docker run -d \
  -p 9000:9000 \
  -p 9443:9443 \
  --name portainer \
  --restart=unless-stopped \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

## Step 4: Fix OCI Image Format Issues

Docker Engine 29+ uses OCI image format by default. Some older images may not have complete OCI metadata:

```bash
# Check image manifest format
docker manifest inspect nginx:latest 2>/dev/null | jq '.mediaType'

# If image was built locally with old Docker:
# Rebuild the image
docker build -t myimage:latest .

# Or pull a fresh copy from Docker Hub
docker pull myimage:latest --platform linux/amd64
```

## Step 5: Fix Multi-Platform Image Issues

After Docker updates, multi-platform image handling changed:

```bash
# Check if the image is multi-platform
docker manifest inspect nginx:latest 2>/dev/null | jq '.manifests[].platform'

# If Portainer shows image details error for multi-platform images:
# Pull the specific platform variant
docker pull --platform linux/amd64 nginx:latest

# Verify
docker inspect nginx:latest | jq '.[0].Architecture'
```

## Step 6: Clear Local Image Cache

```bash
# Remove and re-pull problematic images
docker rmi nginx:latest
docker pull nginx:latest

# Check if image details work after fresh pull
docker inspect nginx:latest | jq '.[0] | {Id, RepoTags, Architecture, Os}'
```

## Step 7: Fix BuildKit Image Metadata Issues

Docker Engine 29+ uses BuildKit by default, which creates images with different metadata structure:

```bash
# Check image history (should work regardless of BuildKit)
docker history myimage:latest

# If history shows missing layers from BuildKit build:
# This is expected — BuildKit uses a different layer structure

# For Portainer to display details correctly:
# Ensure Portainer 2.20+ is installed (it supports BuildKit image format)
```

## Step 8: Fix for Images Built with Docker Compose

```bash
# Images built by docker compose may have incomplete metadata
docker compose build

# Add labels to improve metadata
# In docker-compose.yml:
services:
  myapp:
    build:
      context: .
      labels:
        - "com.example.version=1.0"
        - "com.example.build-date=${BUILD_DATE}"
```

## Step 9: Rebuild the Portainer Snapshot

```bash
# Force Portainer to rebuild its image cache
TOKEN=$(curl -s -X POST http://localhost:9000/api/auth \
  -H "Content-Type: application/json" \
  -d '{"Username":"admin","Password":"yourpassword"}' | jq -r .jwt)

# Trigger a fresh snapshot
curl -X POST \
  -H "Authorization: Bearer $TOKEN" \
  http://localhost:9000/api/endpoints/1/docker/snapshot

# Wait a moment, then refresh Portainer UI
sleep 10
```

## Step 10: Fix Permission Issues for Image Inspection

```bash
# Verify Portainer has access to the Docker images directory
# This is accessed via the Docker socket — not directly

# Test Docker socket works
curl --unix-socket /var/run/docker.sock \
  http://localhost/v1.45/images/json | jq '.[0].RepoTags'

# If socket permissions are wrong:
ls -la /var/run/docker.sock
# Should be: srw-rw---- 1 root docker

# Ensure Portainer container user has socket access
docker inspect portainer | jq '.[0].HostConfig.Binds'
# Should show: /var/run/docker.sock:/var/run/docker.sock
```

## Step 11: Check for Corrupted Image Layers

```bash
# Verify image integrity
docker save nginx:latest | docker load

# Or use docker scan to check for issues
docker scout cves nginx:latest 2>/dev/null | head -10

# Remove and re-pull corrupted images
docker rmi --force nginx:latest
docker pull nginx:latest
```

## Conclusion

"Unable to Retrieve Image Details" after a Docker update is most commonly resolved by updating Portainer to the latest version that supports the new Docker API format. Secondary causes are multi-platform image handling changes (fixed by pulling with explicit `--platform`), OCI format differences (fixed by rebuilding or re-pulling images), and BuildKit metadata changes. Always update Portainer whenever you update Docker Engine to maintain compatibility.
