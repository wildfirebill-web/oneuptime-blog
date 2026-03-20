# How to Fix 'Unable to Retrieve Image Details' After Docker Update

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Troubleshooting, Docker Images, API, Compatibility, Docker Update

Description: Learn how to fix 'Unable to Retrieve Image Details' errors in Portainer that appear after Docker Engine updates due to API version changes or metadata format differences.

---

The "Unable to Retrieve Image Details" error typically appears after upgrading Docker Engine. It occurs when Portainer uses a Docker API call that returned data in one format in the old Docker version but changed fields or structure in the new version.

## Step 1: Check Docker API Version Compatibility

```bash
# Check the Docker daemon API version

docker version --format '{{.Server.APIVersion}}'

# Check what API version Portainer is using
docker logs portainer 2>&1 | grep -i "api version\|api_version"

# Check Docker's minimum supported API version
docker version --format '{{.Server.MinAPIVersion}}'
```

## Step 2: Check the Specific Error in Portainer Logs

```bash
# Find the exact error message with context
docker logs portainer 2>&1 | grep -B 2 -A 5 "image details\|image inspect\|unable to retrieve"
```

## Step 3: Test the Image Inspect API Directly

```bash
# Test Docker image inspection directly
docker inspect <image-name>:<tag>

# If this fails, the issue is in Docker, not Portainer
# If this succeeds but Portainer fails, it's a Portainer API parsing issue
```

## Step 4: Update Portainer to Match Docker Version

Portainer releases are timed to Docker releases. After a Docker update, update Portainer:

```bash
# Pull the latest Portainer
docker pull portainer/portainer-ce:latest

# Stop and remove the current Portainer container
docker stop portainer && docker rm portainer

# Redeploy with the same configuration
docker run -d \
  --name portainer \
  --restart=always \
  -p 9000:9000 -p 9443:9443 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

## Step 5: Clear Image Cache in Portainer

Portainer may be displaying cached image metadata from before the Docker update:

```bash
# Force a full snapshot refresh
docker restart portainer
```

## Step 6: Check for Corrupted Image Layers

Sometimes a Docker update leaves behind corrupted image metadata:

```bash
# Check Docker storage for consistency
docker system df
docker image ls --no-trunc

# Remove and re-pull any image with missing details
docker rmi <image-name>
docker pull <image-name>
```
