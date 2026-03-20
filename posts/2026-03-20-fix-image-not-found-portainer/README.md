# How to Fix 'Image Not Found' Errors When Deploying in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Troubleshooting, Docker Images, Registry, Deployment, Pull Errors

Description: Learn how to diagnose and fix 'image not found' errors in Portainer deployments, covering registry authentication, image tag issues, and private registry configuration.

---

"Image Not Found" errors in Portainer mean Docker cannot pull the specified image. The cause ranges from a simple typo in the image name to missing registry authentication for a private registry.

## Step 1: Verify the Image Name and Tag

```bash
# Test pulling the image directly on the Docker host

docker pull <image-name>:<tag>

# Common mistakes:
# Wrong tag: ubuntu:22 (should be ubuntu:22.04)
# Wrong registry: myregistry.io/app (missing tag, defaults to :latest which may not exist)
# Typo in image name
```

## Step 2: Check if the Image Exists on Docker Hub

```bash
# Search Docker Hub for the image
docker search <image-name>

# Or use the Docker Hub API
curl -s "https://hub.docker.com/v2/repositories/<org>/<image>/tags/?page_size=10" | \
  python3 -m json.tool | grep "name"
```

## Step 3: Handle Private Registry Authentication

For images in private registries, add the registry credentials to Portainer:

1. In Portainer go to **Registries > Add Registry**.
2. Choose **Custom Registry**.
3. Enter the registry URL, username, and password.
4. Click **Add Registry**.

Then in your stack or container definition, reference the image with the full registry path:

```yaml
services:
  app:
    # Always include the full registry URL for private images
    image: registry.example.com/myorg/myapp:1.2.3
```

## Step 4: Fix AWS ECR Authentication

AWS ECR uses temporary tokens that expire every 12 hours. Automate token refresh:

```bash
# Get a fresh ECR login token
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  123456789.dkr.ecr.us-east-1.amazonaws.com

# Add the ECR registry to Portainer with the refreshed token
```

## Step 5: Handle Rate Limiting on Docker Hub

Docker Hub has pull rate limits for unauthenticated users (100 pulls/6h):

```bash
# Check your current rate limit status
TOKEN=$(curl -s "https://auth.docker.io/token?service=registry.docker.io&scope=repository:ratelimitpreview/test:pull" | jq -r .token)
curl -s --head \
  -H "Authorization: Bearer $TOKEN" \
  https://registry-1.docker.io/v2/ratelimitpreview/test/manifests/latest | \
  grep -i ratelimit
```

Add your Docker Hub credentials to Portainer's registry settings to get authenticated pull limits (5000/day).

## Step 6: Check Portainer Network Access

If Portainer is in a network-restricted environment:

```bash
# Test if the Portainer host can reach Docker Hub
docker run --rm alpine ping -c 3 registry-1.docker.io

# Test HTTPS connectivity
docker run --rm alpine sh -c "apk add curl && curl -I https://registry-1.docker.io"
```
