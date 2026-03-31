# How to Monitor Service Image Updates in Portainer on Swarm - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker Swarm, Images, Update, DevOps

Description: Learn how to use Portainer to monitor Docker image updates for Swarm services and automate or trigger image refreshes.

## Introduction

Keeping service images up to date is an ongoing operational concern in Docker Swarm. Portainer provides visibility into which services are running outdated images and tools to update them. This guide covers monitoring image staleness and strategies for keeping your Swarm services current.

## Prerequisites

- Portainer CE or BE on Docker Swarm
- Running Swarm services
- Access to Docker Hub or a private registry from Portainer

## Step 1: Check Current Image Versions

From the Services list, the **Image** column shows the current image and tag for each service:

```text
Service         Image                  Replicas
web-frontend    nginx:alpine           3/3
api-backend     myapp:v2.1             4/4
database        postgres:15-alpine     1/1
```

To see the specific image digest (not just the tag):

1. Click on a service
2. View the **Image** field in the service details
3. The full image reference includes the registry and digest

```bash
# CLI: Check exact image digest in use

docker service inspect web-frontend --format '{{.Spec.TaskTemplate.ContainerSpec.Image}}'
# Output: nginx:alpine@sha256:abc123...
```

## Step 2: Enable Image Update Notifications (Portainer BE)

Portainer Business Edition includes image update notifications:

1. Go to **Settings → Notifications**
2. Enable image update alerts
3. Configure notification channels (email, Slack, webhook)

When a new image is available, Portainer notifies you through the configured channels.

## Step 3: Manual Image Update Check

To check if an image has updates:

```bash
# Pull the latest version of the image
docker pull nginx:alpine

# Compare digests
docker images --digests nginx:alpine
# If the digest differs from what's in the service, an update is available

# Check what the service is running
docker service inspect web-frontend --format '{{.Spec.TaskTemplate.ContainerSpec.Image}}'
```

## Step 4: Update a Service to the Latest Image

### Via Portainer UI

1. Click on the service
2. Click **Edit this service**
3. Optionally change the image tag
4. Click **Update the service**

Portainer uses `--force` update by default, which re-pulls the image.

### Via CLI

```bash
# Force update to pull the latest image with the same tag
docker service update --force web-frontend

# Update to a specific new tag
docker service update --image nginx:1.25-alpine web-frontend

# Update multiple services at once
for svc in web-frontend api-backend; do
    docker service update --force $svc
done
```

## Step 5: Automate Image Update Checks with Watchtower

Deploy Watchtower to automatically update Swarm services when new images are available:

```yaml
# watchtower-swarm.yml
version: "3.8"

services:
  watchtower:
    image: containrrr/watchtower
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      # Check for updates every hour
      - WATCHTOWER_POLL_INTERVAL=3600
      # Send Slack notification on update
      - WATCHTOWER_NOTIFICATION_SLACK_HOOK_URL=${SLACK_WEBHOOK}
      # Only update services with this label
      - WATCHTOWER_LABEL_ENABLE=true
      # Remove old images after update
      - WATCHTOWER_CLEANUP=true
    command: --label-enable --debug
    deploy:
      placement:
        constraints:
          - node.role == manager
```

Label services you want Watchtower to manage:

```yaml
services:
  web:
    image: nginx:alpine
    labels:
      - com.centurylinklabs.watchtower.enable=true  # Watchtower will auto-update this
```

## Step 6: Use Image Tags vs Digests

### Using Tags (Flexible)

```yaml
services:
  web:
    image: nginx:alpine    # Updates when nginx:alpine changes
```

Tags like `latest` or `alpine` float - they can point to new image digests.

### Using Digests (Immutable)

```yaml
services:
  web:
    image: nginx:alpine@sha256:abc123def456...  # Always this exact image
```

Using digests ensures reproducibility but requires explicit updates.

### Pinned Semver Tags (Best Practice)

```yaml
services:
  web:
    image: nginx:1.25-alpine     # Specific version, still updated with patches
  api:
    image: myapp:v2.1.3          # Fully pinned to a release
```

## Step 7: Image Update Policy for Production

Implement a structured update process:

```bash
#!/bin/bash
# update-service-images.sh
# Run monthly to update production services

SERVICES=(web-frontend api-backend)
LOG_FILE="/var/log/image-updates.log"

for svc in "${SERVICES[@]}"; do
    echo "$(date): Checking $svc" >> $LOG_FILE

    # Get current image
    CURRENT=$(docker service inspect $svc --format '{{.Spec.TaskTemplate.ContainerSpec.Image}}')

    # Force update (pulls latest, only updates if digest changed)
    docker service update --force $svc

    # Get new image
    NEW=$(docker service inspect $svc --format '{{.Spec.TaskTemplate.ContainerSpec.Image}}')

    if [ "$CURRENT" != "$NEW" ]; then
        echo "$(date): $svc updated: $CURRENT → $NEW" >> $LOG_FILE
    else
        echo "$(date): $svc unchanged" >> $LOG_FILE
    fi
done
```

## Conclusion

Monitoring and managing image updates for Swarm services is an important part of cluster maintenance. Portainer gives you visibility into current image versions, and tools like Watchtower can automate the update process. For production systems, implement a structured update policy that balances keeping images current with the stability needed for reliable operations.
