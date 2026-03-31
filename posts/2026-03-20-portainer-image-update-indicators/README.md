# How to Identify Image Update Indicators in Portainer - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Image, Update, DevOps

Description: Learn how to use Portainer's image update indicators to detect when newer versions of your container images are available in the registry.

## Introduction

Keeping container images up to date is important for security patches and new features. Portainer can check whether newer versions of your images are available on Docker Hub or private registries and show visual indicators in the UI. This guide covers how to use this feature and automate image update checks.

## Prerequisites

- Portainer installed with a connected Docker environment
- Containers running and connected to registries

## Step 1: Enable Image Update Checking in Portainer

Portainer can periodically check for image updates:

1. Navigate to **Settings** in Portainer.
2. Look for **Image update checking** or **Registry settings**.
3. Enable **Check for image updates**.
4. Set the check frequency (e.g., daily).

## Step 2: Identify the Update Indicator

In the Portainer container list, containers with newer image versions available show a visual indicator:

- A **cloud icon** or **upgrade arrow** appears next to the container.
- Hovering shows "An update is available for this container."

In the **Images** view, images with available updates show similar indicators.

## Step 3: Manual Update Check

To manually check for updates:

```bash
# Docker CLI: check if a new image digest is available

# Compare local digest vs. registry digest

# Get local image digest:
docker inspect nginx:alpine --format '{{.Id}}'
# sha256:abc123...

# Get registry digest (requires network request):
docker pull --quiet nginx:alpine
# If output shows "Status: Downloaded newer image" → update available
# If output shows "Status: Image is up to date" → no update needed
```

## Step 4: Automated Image Update Check Script

```bash
#!/bin/bash
# check-image-updates.sh
# Checks if updates are available for all running container images

echo "=== Docker Image Update Check: $(date) ==="
echo ""

# Get list of images used by running containers
declare -A CHECKED_IMAGES

docker ps --format "{{.Image}}" | sort -u | while read image; do
    # Skip already checked images
    [ "${CHECKED_IMAGES[$image]}" == "1" ] && continue
    CHECKED_IMAGES[$image]=1

    # Get local image ID
    LOCAL_ID=$(docker inspect "${image}" --format '{{.Id}}' 2>/dev/null)

    # Pull and check if a new image is available
    PULL_OUTPUT=$(docker pull "${image}" 2>&1)

    if echo "${PULL_OUTPUT}" | grep -q "Downloaded newer image"; then
        echo "⬆️  UPDATE AVAILABLE: ${image}"
        echo "   Local:    ${LOCAL_ID:0:12}..."
        NEW_ID=$(docker inspect "${image}" --format '{{.Id}}' 2>/dev/null)
        echo "   Registry: ${NEW_ID:0:12}..."
    elif echo "${PULL_OUTPUT}" | grep -q "Image is up to date"; then
        echo "✓  Up to date: ${image}"
    else
        echo "✗  Check failed: ${image}"
    fi
done
```

## Step 5: Using Watchtower for Automatic Updates

Deploy Watchtower to automatically update containers when new images are available:

```yaml
# watchtower-stack.yml
version: "3.8"

services:
  watchtower:
    image: containrrr/watchtower:latest
    restart: unless-stopped
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      # Check for updates every 6 hours (21600 seconds)
      - WATCHTOWER_SCHEDULE=0 0 */6 * * *
      # Send notification to Slack
      - WATCHTOWER_NOTIFICATION_SLACK_HOOK_URL=${SLACK_WEBHOOK}
      # Only notify, don't actually update (dry run):
      - WATCHTOWER_MONITOR_ONLY=true   # Remove for actual updates
      # Clean up old images after update:
      - WATCHTOWER_CLEANUP=true
      # Log level:
      - WATCHTOWER_DEBUG=false
```

With `WATCHTOWER_MONITOR_ONLY=true`, Watchtower only notifies about updates without applying them - safer for production.

## Step 6: Portainer's Automatic Update for Stacks

For stacks, enable Git-based auto-updates in Portainer:

1. Navigate to **Stacks**.
2. Click your stack name.
3. Enable **Auto update**.
4. Choose **Polling** (check every N minutes) or **Webhook**.
5. Enable **Force re-pull image** to pull new image versions on each update.

This ensures Portainer re-pulls images whenever the stack is updated.

## Step 7: Image Update Notification Pattern

For production environments, notify before updating:

```bash
#!/bin/bash
# notify-image-updates.sh
# Checks for updates and sends notifications

SLACK_WEBHOOK="${SLACK_WEBHOOK_URL}"

send_notification() {
    local message="$1"
    curl -s -X POST "${SLACK_WEBHOOK}" \
        -H "Content-Type: application/json" \
        -d "{\"text\": \"${message}\"}"
}

docker ps --format "{{.Names}}\t{{.Image}}" | while IFS=$'\t' read name image; do
    LOCAL_ID=$(docker inspect "${image}" --format '{{.Id}}' 2>/dev/null | cut -c1-12)

    # Pull image silently
    PULL_OUTPUT=$(docker pull "${image}" 2>&1)
    NEW_ID=$(docker inspect "${image}" --format '{{.Id}}' 2>/dev/null | cut -c1-12)

    if [ "${LOCAL_ID}" != "${NEW_ID}" ]; then
        send_notification "🔔 Image update available for container *${name}*\nImage: \`${image}\`\nOld: ${LOCAL_ID}\nNew: ${NEW_ID}"
    fi
done
```

## Step 8: Digest-Based Update Detection

For more reliable update detection using image digests:

```bash
#!/bin/bash
# digest-check.sh
# More reliable update detection using content-addressable digests

check_update() {
    local image="$1"

    # Get the image's current digest
    LOCAL_DIGEST=$(docker inspect --format '{{index .RepoDigests 0}}' "${image}" 2>/dev/null)

    # Get the registry digest without pulling
    REGISTRY_DIGEST=$(docker manifest inspect "${image}" 2>/dev/null | \
        jq -r '.config.digest' 2>/dev/null)

    if [ -z "${LOCAL_DIGEST}" ] || [ -z "${REGISTRY_DIGEST}" ]; then
        echo "Could not determine digest for: ${image}"
        return
    fi

    # Extract just the digest portion for comparison
    LOCAL_SHORT=$(echo "${LOCAL_DIGEST}" | cut -d@ -f2 | cut -c1-12)
    REGISTRY_SHORT=$(echo "${REGISTRY_DIGEST}" | cut -c1-12)

    if [ "${LOCAL_SHORT}" != "${REGISTRY_SHORT}" ]; then
        echo "UPDATE AVAILABLE: ${image}"
    else
        echo "Up to date: ${image}"
    fi
}

for image in $(docker ps --format "{{.Image}}" | sort -u); do
    check_update "${image}"
done
```

## Best Practices for Image Updates

- **Don't auto-update production containers** without testing - use `WATCHTOWER_MONITOR_ONLY=true`.
- **Update on a schedule** (e.g., nightly in staging, weekly in production).
- **Use semantic versioning** - pin to `v2.1` (gets patch updates) not `latest` (gets breaking changes).
- **Test updates in staging** before applying to production.
- **Keep a rollback plan** - know the previous image digest before updating.

## Conclusion

Portainer's image update indicators and automated checking tools help you stay current with security patches and new releases. For production environments, use notification-only mode to be aware of updates before they're applied. Combine with CI/CD pipelines to test updates in staging before promoting to production, ensuring you get the benefits of updates without unexpected disruptions.
