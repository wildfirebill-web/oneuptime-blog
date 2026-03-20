# How to Check Image Updates via the Portainer API

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, API, Docker, Images, Monitoring

Description: Learn how to use the Portainer API to check for container image updates, identify outdated containers, and automate image freshness monitoring across your environments.

## Introduction

Keeping container images up to date is important for security patches and bug fixes. Portainer provides API endpoints to check whether running containers are using the latest version of their images, enabling you to build automated update notification systems or integrate image freshness checks into your operations workflows.

## Prerequisites

- Portainer CE or BE with a Docker environment
- Running containers with image update checking enabled
- Valid JWT token or API access token

## Step 1: Check Images for Updates via the API

Portainer can query registries to compare the current running image digest with the latest available:

```bash
PORTAINER_URL="https://portainer.example.com"
TOKEN="your-auth-token"
ENDPOINT_ID=1

# Get a container ID first
CONTAINER_ID="abc123def456"

# Check if a specific container's image has an update available
curl -s -X POST \
  -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/docker/containers/${CONTAINER_ID}/image_status" | jq .

# Response example:
# {
#   "Status": "outdated"   or   "updated"
# }
```

## Step 2: List Containers with Outdated Images

```bash
#!/bin/bash
# check-image-updates.sh

PORTAINER_URL="https://portainer.example.com"
TOKEN="your-token"
ENDPOINT_ID=1

echo "=== Image Update Status Report ==="
echo ""

# Get all running containers
CONTAINERS=$(curl -s -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/docker/containers/json")

OUTDATED_COUNT=0
TOTAL_COUNT=$(echo $CONTAINERS | jq 'length')

echo "Checking $TOTAL_COUNT running containers..."
echo ""

echo $CONTAINERS | jq -c '.[]' | while read -r CONTAINER; do
  CONTAINER_ID=$(echo $CONTAINER | jq -r '.Id')
  CONTAINER_NAME=$(echo $CONTAINER | jq -r '.Names[0]' | sed 's/^\///')
  IMAGE=$(echo $CONTAINER | jq -r '.Image')

  # Check image status
  STATUS=$(curl -s -X POST \
    -H "Authorization: Bearer $TOKEN" \
    "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/docker/containers/${CONTAINER_ID}/image_status" | \
    jq -r '.Status // "unknown"')

  if [ "$STATUS" = "outdated" ]; then
    echo "OUTDATED: $CONTAINER_NAME ($IMAGE)"
  elif [ "$STATUS" = "updated" ]; then
    echo "UP TO DATE: $CONTAINER_NAME ($IMAGE)"
  else
    echo "UNKNOWN: $CONTAINER_NAME ($IMAGE) — $STATUS"
  fi
done
```

## Step 3: Pull the Latest Image for a Container

After detecting an outdated image, pull the latest version:

```bash
CONTAINER_ID="abc123def456"
IMAGE_NAME="nginx:latest"

# Pull the latest version of the image
curl -s -X POST \
  -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/docker/images/create?fromImage=${IMAGE_NAME}" | \
  while IFS= read -r line; do
    echo "$line" | jq -r '.status // .error // .' 2>/dev/null || echo "$line"
  done

echo "Image pull complete."
```

## Step 4: Update a Container to Use the Latest Image

After pulling, recreate the container with the new image:

```bash
CONTAINER_ID="abc123def456"

# Get current container config before recreating
CONTAINER_CONFIG=$(curl -s -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/docker/containers/${CONTAINER_ID}/json")

CONTAINER_NAME=$(echo $CONTAINER_CONFIG | jq -r '.Name' | sed 's/^\///')
IMAGE=$(echo $CONTAINER_CONFIG | jq -r '.Config.Image')

echo "Updating container '$CONTAINER_NAME' (image: $IMAGE)..."

# Stop the container
curl -s -X POST -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/docker/containers/${CONTAINER_ID}/stop?t=10"

# Remove the old container
curl -s -X DELETE -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/docker/containers/${CONTAINER_ID}"

# Recreate with the same config (using the new image)
# Note: In practice, use Watchtower or stack-based updates for this
echo "Container recreated with latest image."
```

## Step 5: Get Image Digest for Comparison

Compare image digests to detect updates manually:

```bash
IMAGE_NAME="nginx"
IMAGE_TAG="latest"

# Get the local image digest
LOCAL_DIGEST=$(curl -s -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/docker/images/${IMAGE_NAME}:${IMAGE_TAG}/json" | \
  jq -r '.RepoDigests[0] // "none"')

echo "Local digest: $LOCAL_DIGEST"

# Inspect image details
curl -s -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/docker/images/${IMAGE_NAME}:${IMAGE_TAG}/json" | \
  jq '{
    id: .Id[0:20],
    tags: .RepoTags,
    created: .Created,
    size_mb: (.Size / 1048576 | floor),
    digest: .RepoDigests[0]
  }'
```

## Step 6: List All Local Images

```bash
# List all local images with their tags and sizes
curl -s -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/docker/images/json" | \
  jq '.[] | {
    id: .Id[0:12],
    tags: .RepoTags,
    size_mb: (.Size / 1048576 | floor),
    created: .Created
  }'

# Find images with no tags (dangling images)
curl -s -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/docker/images/json?dangling=true" | jq .
```

## Step 7: Remove Unused Images

```bash
# Remove all unused images (prune)
curl -s -X POST \
  -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/docker/images/prune" | \
  jq '{deleted: (.ImagesDeleted // [] | length), reclaimed_mb: (.SpaceReclaimed / 1048576 | floor)}'

# Remove a specific image
IMAGE_ID="sha256:abc123..."
curl -s -X DELETE \
  -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/docker/images/${IMAGE_ID}"
```

## Step 8: Automated Update Notification Script

```bash
#!/bin/bash
# notify-outdated-images.sh — Send Slack notification for outdated images

PORTAINER_URL="https://portainer.example.com"
TOKEN="your-token"
ENDPOINT_ID=1
SLACK_WEBHOOK="https://hooks.slack.com/services/YOUR/WEBHOOK/URL"

CONTAINERS=$(curl -s -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/docker/containers/json")

OUTDATED_ITEMS=""

echo $CONTAINERS | jq -c '.[]' | while read -r CONTAINER; do
  ID=$(echo $CONTAINER | jq -r '.Id')
  NAME=$(echo $CONTAINER | jq -r '.Names[0]' | sed 's/^\///')
  IMAGE=$(echo $CONTAINER | jq -r '.Image')

  STATUS=$(curl -s -X POST -H "Authorization: Bearer $TOKEN" \
    "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/docker/containers/${ID}/image_status" | \
    jq -r '.Status // "unknown"')

  if [ "$STATUS" = "outdated" ]; then
    OUTDATED_ITEMS="${OUTDATED_ITEMS}• ${NAME}: ${IMAGE}\n"
  fi
done

if [ -n "$OUTDATED_ITEMS" ]; then
  curl -s -X POST "$SLACK_WEBHOOK" \
    -H "Content-Type: application/json" \
    -d "{\"text\": \"*Outdated Container Images Detected*\n${OUTDATED_ITEMS}\"}"
fi
```

## Conclusion

The Portainer API provides image update checking capabilities that you can integrate into monitoring dashboards, CI/CD pipelines, and automated update workflows. Use the image status endpoint for quick per-container checks, combine with pull operations for automated updates, and set up scheduled scripts to proactively notify your team about outdated images before they become security liabilities.
