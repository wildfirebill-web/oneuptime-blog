# How to Recreate a Container with Updated Settings in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Containers, Operations, DevOps

Description: Learn how to recreate a Docker container in Portainer to apply configuration changes or update to a new image version without losing your setup.

## Introduction

When you need to change a container's configuration — environment variables, port mappings, volume mounts, or image version — Docker requires creating a new container. Portainer's **Duplicate/Edit** and **Recreate** workflows make this straightforward. This guide explains how to safely recreate containers with updated settings.

## Prerequisites

- Portainer installed with a connected Docker environment
- A running container to recreate

## Why Recreate Is Needed

Unlike virtual machines, you cannot modify most Docker container settings while it's running. To apply changes:

- Environment variables → Recreate
- Port mappings → Recreate
- Volume mounts → Recreate
- Resource limits → Recreate
- Image version → Recreate
- Restart policy → Recreate
- Labels → Recreate

The only things you can change without recreation:
- Run exec commands inside the container
- Modify files inside the container (non-persistent)

## Method 1: Edit and Recreate (Portainer's Built-in Flow)

### Step 1: Open Container Settings

1. Navigate to **Containers** in Portainer.
2. Click on the container name.
3. Click **Duplicate/Edit** button.

### Step 2: Modify the Settings

The form opens pre-filled with current settings. Change whatever needs updating:

```
Original image:  myorg/myapp:v2.0.0
Updated image:   myorg/myapp:v2.1.0

Original env:    LOG_LEVEL=info
Updated env:     LOG_LEVEL=warn

Original port:   8080 → 80
Updated port:    8080 → 80 (unchanged)
```

### Step 3: Deploy the Updated Container

Scroll to the bottom and click **Deploy the container**.

Important: This creates a NEW container, but with the same name. Docker will fail if the old container is still running with that name:

1. First stop the old container.
2. Then deploy the new one.

Or use the **Replace** option if Portainer prompts you to replace the existing container.

## Method 2: Pull New Image and Recreate (Image Update)

For image version updates:

1. Navigate to **Images**.
2. Pull the new image version.
3. Navigate to **Containers**.
4. Click the container name.
5. Click **Duplicate/Edit**.
6. Update the image tag.
7. Stop the old container.
8. Deploy the new container.

Or use Portainer's image update indicator:

1. In the container list, look for an **update available** badge.
2. Click it to trigger an automatic pull + recreate.

## Method 3: Blue-Green Deployment (Zero Downtime)

For production containers, avoid downtime with a blue-green approach:

```bash
#!/bin/bash
# blue-green-deploy.sh
# Deploys new version with zero downtime using Portainer API

PORTAINER_URL="${PORTAINER_URL}"
API_KEY="${PORTAINER_API_KEY}"
ENDPOINT_ID="${PORTAINER_ENDPOINT_ID:-1}"

# Create new "green" container with updated image
curl -X POST \
  -H "X-API-Key: ${API_KEY}" \
  -H "Content-Type: application/json" \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/docker/containers/create?name=web-app-green" \
  -d '{
    "Image": "myorg/myapp:v2.1.0",
    "ExposedPorts": {"8080/tcp": {}},
    "HostConfig": {
      "PortBindings": {"8080/tcp": [{"HostPort": "8081"}]},
      "RestartPolicy": {"Name": "unless-stopped"}
    }
  }' | jq .

# Start green container
# (test it on port 8081)
# Switch load balancer to green
# Stop blue container (old version on port 8080)
# Remove blue container
```

## Method 4: Docker Compose Stack Redeploy

For containers managed as Portainer stacks, updating is simpler:

1. Navigate to **Stacks**.
2. Click the stack name.
3. Click **Editor** to view/edit the compose file.
4. Update the image tag or configuration.
5. Click **Update the stack**.

```yaml
# Updated stack — change the image version
services:
  web:
    image: myorg/webapp:v2.1.0  # Changed from v2.0.0
    restart: unless-stopped
    environment:
      - LOG_LEVEL=warn  # Changed from info
    ports:
      - "8080:80"
```

Portainer will stop the old containers and start new ones with the updated config.

## Step 5: Handling Named Volumes During Recreation

Named volumes persist through container recreation:

```yaml
services:
  app:
    image: myorg/app:v2.1.0
    volumes:
      # Named volume — data persists through recreation
      - app_data:/app/data
      # Bind mount — always points to host path
      - /etc/app/config:/app/config:ro

volumes:
  app_data:  # Data here survives container removal/recreation
```

This means you can safely recreate the container — your data in named volumes is untouched.

## Step 6: Verify After Recreation

After recreating:

1. Check the container is **Running** in Portainer.
2. Verify the new settings took effect:

```bash
# Verify image version:
docker inspect my-container | jq '.[].Config.Image'

# Verify environment variables:
docker exec my-container env | grep LOG_LEVEL

# Verify the app is working:
curl http://localhost:8080/health
```

## Best Practices

- **Test recreations in staging first** — validate the new settings work before production.
- **Keep volume names the same** when recreating — data persists automatically.
- **Use Portainer stacks** for multi-container applications — stack updates handle recreation automatically.
- **Document configuration changes** — add comments in compose files.
- **Automate recreations** via Portainer webhooks for CI/CD pipelines.

## Conclusion

Recreating containers in Portainer is the correct approach for applying configuration changes and image updates. The key is to ensure named volumes preserve your data across recreations. For multi-container applications, use Portainer Stacks for a cleaner update workflow, and for production environments, consider blue-green deployments to minimize downtime during updates.
