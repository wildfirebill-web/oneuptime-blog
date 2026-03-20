# How to Force Pull Latest Images When Updating Stacks in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Stacks, Images, DevOps

Description: Learn how to force Portainer to pull the latest version of Docker images when updating stacks, ensuring mutable tags like latest always use fresh images.

## Introduction

Docker caches images locally to speed up deployments. When you update a stack, Docker only pulls an image if its tag is not already present locally — meaning `myapp:latest` will not be re-pulled if that tag already exists on the host, even if the upstream image was updated. For stacks using mutable tags (like `latest`, `main`, or `stable`), this behavior prevents getting the newest image. Portainer's **Force re-pull images** option and CLI techniques solve this problem.

## Prerequisites

- Portainer with an existing deployed stack
- Understanding of Docker image tag mutability

## Why Force Pull Is Needed

```
Scenario:
1. You build and push myorg/api:latest to Docker Hub
2. Docker host already has myorg/api:latest cached
3. Stack update runs — Docker sees tag exists, skips pull
4. Old image is still running — your update is NOT deployed!

Fix: Force pull ensures Docker always contacts the registry
     and downloads the latest digest for the tag
```

## Step 1: Force Pull During Stack Update (Portainer UI)

When updating an existing stack:

1. Navigate to **Stacks** → click the stack name.
2. Make any changes to the Compose YAML or environment variables.
3. Find the **Update** section below the editor.
4. Check **Re-pull image** (or **Force re-pull images**).
5. Click **Update the stack**.

Portainer will run `docker pull` for every image in the stack before recreating containers.

## Step 2: Force Pull with Git-Based Stacks

For Git-based stacks with automatic updates:

1. Navigate to **Stacks** → click the stack name.
2. In the **Automatic updates** section.
3. Enable **Force re-pull images**.
4. This applies on every auto-update cycle (both polling and webhook).

## Step 3: Force Pull via CLI

```bash
# Pull the latest image manually:
docker pull myorg/api:latest

# Then restart the container to use the new image:
docker compose pull && docker compose up -d

# For a specific service only:
docker compose pull api
docker compose up -d api

# For Swarm services:
docker service update --image myorg/api:latest my-stack_api
# This forces a pull and rolling update
```

## Step 4: Use Image Digests for Certainty

For production, use immutable image digests instead of mutable tags:

```yaml
# Mutable tag — might not get re-pulled:
services:
  api:
    image: myorg/api:latest

# Immutable digest — always points to exact image:
services:
  api:
    image: myorg/api@sha256:abc123def456789...
```

Get the digest after building:

```bash
# After docker push:
docker image inspect myorg/api:latest --format '{{index .RepoDigests 0}}'
# myorg/api@sha256:abc123def456789...

# Use this digest in docker-compose.yml to pin the exact image
```

## Step 5: Update Image Tags via CI/CD

The best practice for production is to update the image tag in the Compose file (or environment variable) on each deployment:

```yaml
# docker-compose.yml — uses IMAGE_TAG variable:
services:
  api:
    image: myorg/api:${IMAGE_TAG:-latest}
```

In your CI/CD pipeline (GitHub Actions example):

```yaml
- name: Deploy new image version
  env:
    PORTAINER_URL: ${{ secrets.PORTAINER_URL }}
    PORTAINER_TOKEN: ${{ secrets.PORTAINER_TOKEN }}
    STACK_ID: ${{ secrets.STACK_ID }}
  run: |
    # Update the IMAGE_TAG env var in Portainer:
    curl -X PUT \
      "${PORTAINER_URL}/api/stacks/${STACK_ID}" \
      -H "X-API-Key: ${PORTAINER_TOKEN}" \
      -H "Content-Type: application/json" \
      -d "{
        \"Env\": [
          {\"name\": \"IMAGE_TAG\", \"value\": \"${{ github.sha }}\"}
        ],
        \"Prune\": false,
        \"PullImage\": true
      }"
```

## Step 6: Verify the Image Was Updated

After a force pull update, confirm the new image is running:

```bash
# Check image ID/digest of the running container:
docker inspect my-stack_api_1 --format '{{.Image}}'
# sha256:abc123... (the image digest)

# Compare with the latest pulled image:
docker image inspect myorg/api:latest --format '{{.Id}}'

# Check when the image was pulled:
docker image inspect myorg/api:latest --format '{{.Metadata.LastTagTime}}'

# View image history to confirm it's the latest build:
docker history myorg/api:latest | head -3
```

## Step 7: Automate Forced Updates with Watchtower

For automatic image updates without manual intervention:

```yaml
# docker-compose.yml — add Watchtower to auto-update other containers
services:
  watchtower:
    image: containrrr/watchtower:latest
    restart: unless-stopped
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    command: --interval 300 --cleanup   # Check every 5 minutes, remove old images
    environment:
      - WATCHTOWER_POLL_INTERVAL=300
```

Note: Watchtower is powerful but can cause unexpected restarts in production. Use with caution and prefer explicit image tag updates via CI/CD for production services.

## Conclusion

Force pulling images in Portainer ensures mutable tags like `latest` always deploy the newest version of your application. Use the **Re-pull image** checkbox during manual stack updates and enable **Force re-pull images** for automatic Git-based updates. For production environments, the most reliable approach is to use specific image tags (version numbers or commit SHAs) that change with each build, making the image version explicit in your Compose file and eliminating reliance on mutable tags.
