# How to Check Image Updates via the Portainer API

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, API, Image Updates, Container Management, Automation

Description: Learn how to use the Portainer API to check for available container image updates across your environments.

## Overview

Portainer can check if newer versions of container images are available from the registry. This is useful for automated workflows that need to identify stale images without manually checking each container.

## Triggering an Image Update Check

```bash
# Trigger Portainer to re-fetch image information for an endpoint
curl -X POST \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/docker/images/create?fromImage=registry.mycompany.com/myapp:latest" \
  -H "Authorization: Bearer ${API_TOKEN}" \
  -H "X-Registry-Auth: $(echo '{"username":"user","password":"pass"}' | base64)"
```

## Listing Images and Their Details

```bash
# List all local images
curl -s "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/docker/images/json" \
  -H "Authorization: Bearer ${API_TOKEN}" | \
  jq '[.[] | {
    id: .Id[7:19],
    repo: .RepoTags[0],
    created: .Created,
    size_mb: (.Size / 1048576 | floor)
  }]'
```

## Checking if a Specific Image Has Updates

```bash
#!/bin/bash
# Check if a local image digest matches the registry digest

PORTAINER_URL="https://portainer.mycompany.com"
API_TOKEN="${PORTAINER_API_TOKEN}"
ENDPOINT_ID=1
IMAGE="nginx:latest"

# Get local image digest
LOCAL_DIGEST=$(curl -s \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/docker/images/${IMAGE}/json" \
  -H "Authorization: Bearer ${API_TOKEN}" | \
  jq -r '.RepoDigests[0]' | cut -d@ -f2)

echo "Local digest: ${LOCAL_DIGEST}"

# Get registry digest (requires docker manifest inspect or registry API)
REGISTRY_DIGEST=$(docker manifest inspect nginx:latest 2>/dev/null | \
  jq -r '.config.digest')

echo "Registry digest: ${REGISTRY_DIGEST}"

if [ "$LOCAL_DIGEST" = "$REGISTRY_DIGEST" ]; then
  echo "Image is up to date"
else
  echo "Update available!"
fi
```

## Using Portainer's Stack Update with Pull

The most common pattern is to trigger a stack redeploy with `pullImage: true`:

```bash
# Redeploy a stack, pulling the latest image version
STACK_ID=3

curl -X PUT "${PORTAINER_URL}/api/stacks/${STACK_ID}?endpointId=${ENDPOINT_ID}" \
  -H "Authorization: Bearer ${API_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "StackFileContent": "...",
    "PullImage": true,
    "Prune": false
  }'
```

## Automated Daily Image Update Check Script

```bash
#!/bin/bash
# daily-image-check.sh - Check all containers for image updates

PORTAINER_URL="https://portainer.mycompany.com"
API_TOKEN="${PORTAINER_API_TOKEN}"
ENDPOINT_ID=1
SLACK_WEBHOOK="${SLACK_WEBHOOK_URL}"

# Get all running containers
CONTAINERS=$(curl -s \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/docker/containers/json" \
  -H "Authorization: Bearer ${API_TOKEN}")

# Check each container's image
OUTDATED=()

while IFS= read -r container; do
  IMAGE=$(echo "$container" | jq -r '.Image')
  NAME=$(echo "$container" | jq -r '.Names[0]')

  # Pull the latest and compare
  PULL_RESULT=$(curl -s -X POST \
    "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/docker/images/create?fromImage=${IMAGE}" \
    -H "Authorization: Bearer ${API_TOKEN}")

  # Check if update was available
  if echo "$PULL_RESULT" | grep -q '"status":"Pull complete"'; then
    OUTDATED+=("${NAME} (${IMAGE})")
  fi
done < <(echo "$CONTAINERS" | jq -c '.[]')

# Notify if updates are available
if [ ${#OUTDATED[@]} -gt 0 ]; then
  MESSAGE="Container image updates available: $(printf '\n- %s' "${OUTDATED[@]}")"
  curl -s -X POST "${SLACK_WEBHOOK}" \
    -H "Content-Type: application/json" \
    -d "{\"text\": \"${MESSAGE}\"}"
fi
```

## Conclusion

Checking for image updates via the Portainer API lets you build automated notification and update workflows. Combine it with a scheduled cron job and notification channels to stay on top of available security patches and new releases.
