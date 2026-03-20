# How to Edit an Existing Stack in Portainer - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Stacks, Docker Compose, DevOps

Description: Learn how to modify and update existing Docker Compose stacks in Portainer, including adding services, changing images, and updating environment variables.

## Introduction

Once a stack is deployed in Portainer, you will inevitably need to modify it - adding a new service, updating an image version, changing environment variables, adjusting resource limits, or restructuring the network configuration. Portainer provides a built-in editor for stacks deployed via the web editor, and update controls for Git-based stacks. Understanding how Portainer applies updates helps you make changes safely without unnecessary downtime.

## Prerequisites

- Portainer with at least one deployed stack
- Understanding of Docker Compose update behavior

## How Stack Updates Work

When you update a stack in Portainer:
1. Docker Compose compares the new definition to the running state.
2. Only changed services are recreated (services with no changes are left running).
3. Networks and volumes are created if new, but existing ones are not removed automatically.
4. The update is applied container by container within each service.

## Step 1: Open the Stack Editor

1. Navigate to **Stacks** in Portainer.
2. Click the stack name you want to edit.
3. The stack detail page shows the current Compose content.
4. Click **Editor** tab to view and edit the Compose YAML.

## Step 2: Add a New Service

To add a Redis cache service to an existing stack:

```yaml
# Add this service block to the existing Compose YAML:

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    command: ["redis-server", "--appendonly", "yes"]
    networks:
      - backend     # Must match an existing network
    volumes:
      - redis_data:/data

# Also add to the volumes section:
volumes:
  postgres_data:    # (existing)
  redis_data:       # (new)
```

After adding, click **Update the stack** - only the new `redis` container is created. Existing containers continue running.

## Step 3: Update a Service Image

To upgrade nginx from `alpine` to a specific version:

```yaml
# Before:
  nginx:
    image: nginx:alpine

# After:
  nginx:
    image: nginx:1.25-alpine
```

After clicking **Update the stack**, only the `nginx` container is recreated with the new image.

To always pull the latest tag:
1. Check **Re-pull image** before clicking Update.
2. This forces a fresh `docker pull` even if the tag hasn't changed.

## Step 4: Modify Environment Variables

Environment variables can be updated two ways:

**Method 1: In the Compose YAML** (for non-sensitive values):
```yaml
services:
  api:
    environment:
      - LOG_LEVEL=debug      # Changed from info to debug
      - WORKERS=4            # Added new variable
```

**Method 2: In the Environment Variables section** (for secrets):
1. In the stack detail page, scroll below the editor.
2. Add, edit, or remove environment variables.
3. Click **Update the stack**.

## Step 5: Change Resource Limits

Add or update resource constraints:

```yaml
# Before (no limits):
  api:
    image: myorg/api:latest

# After (with limits):
  api:
    image: myorg/api:latest
    deploy:
      resources:
        limits:
          cpus: "0.5"
          memory: 512M
        reservations:
          memory: 256M
```

For standalone Docker (not Swarm), use:
```yaml
  api:
    image: myorg/api:latest
    mem_limit: 512m
    cpus: 0.5
```

## Step 6: Rename a Service

Renaming a service effectively creates a new container - Docker Compose tracks services by name:

```yaml
# Before:
  app:
    image: myorg/app:latest

# After (renamed):
  api:
    image: myorg/app:latest
```

After updating: Docker Compose creates `api` container and removes `app` container. Data in named volumes is preserved.

## Step 7: Roll Back to Previous Configuration

Portainer does not natively maintain a stack version history. To enable rollback:

```bash
# Keep a backup before editing:
# In Portainer, copy the current Compose YAML before making changes
# Paste it into a file:
cat > stack-backup-$(date +%Y%m%d).yml << 'EOF'
(paste current content here)
EOF
```

For Git-based stacks:
- Changes to the repo are tracked by commit history.
- To rollback, point the stack to a previous commit SHA.
- In Portainer: **Stacks** → click stack → set **Repository ref** to the previous commit.

## Conclusion

Editing stacks in Portainer is straightforward - open the editor, modify the Compose YAML, and click Update. Docker Compose intelligently applies only the changes: new services are created, changed services are recreated, and unchanged services continue running without interruption. For production stacks, use Git-based deployment so every edit is a commit, enabling rollback to any previous state. Before making significant changes, copy the current Compose YAML as a backup or ensure your Git history is up to date.
