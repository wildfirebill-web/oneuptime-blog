# How to Remove a Stack in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Stack, Cleanup, DevOps

Description: Learn how to safely remove Docker Compose stacks in Portainer, understanding what gets deleted and how to preserve important data.

## Introduction

Removing a stack in Portainer stops and removes all containers, networks, and (optionally) volumes associated with that stack. Understanding exactly what gets removed - and what doesn't - is critical to avoid accidental data loss. Portainer's stack removal is equivalent to running `docker compose down` with configurable options for volume handling.

## Prerequisites

- Portainer with at least one deployed stack
- Understanding of which data you need to preserve

## What Gets Removed (and What Doesn't)

| Resource | Removed by default | With --volumes flag |
|----------|--------------------|---------------------|
| Containers | Yes | Yes |
| Stack networks | Yes | Yes |
| Named volumes (in Compose) | **No** | Yes |
| Bind mounts (host paths) | No (files remain on host) | No |
| Images | No | No |
| Portainer stack metadata | Yes | Yes |

By default, named volumes persist after stack removal to prevent accidental data loss. This is intentional - you must explicitly choose to remove volumes.

## Step 1: Back Up Volumes Before Removal

Before removing a stack, back up any important data:

```bash
# Identify volumes used by the stack:

docker compose ps -q | xargs docker inspect \
  --format '{{.Name}}: {{range .Mounts}}{{.Name}} {{end}}'

# Backup a volume:
docker run --rm \
  -v myapp_postgres_data:/source:ro \
  -v /backup:/backup \
  alpine tar czf /backup/postgres_data_backup_$(date +%Y%m%d_%H%M%S).tar.gz \
  -C /source .

echo "Backup complete: $(ls -lh /backup/)"
```

## Step 2: Remove the Stack via Portainer UI

### Remove Without Deleting Volumes (Safe - Default)

1. Navigate to **Stacks** in Portainer.
2. Check the checkbox next to the stack.
3. Click **Remove**.
4. In the confirmation dialog:
   - Leave **Remove associated volumes** **unchecked** (default).
5. Click **Remove**.

Result: containers and networks are removed, volumes remain for data preservation.

### Remove Including Volumes (Destructive)

1. Navigate to **Stacks** in Portainer.
2. Check the checkbox next to the stack.
3. Click **Remove**.
4. In the confirmation dialog:
   - Check **Remove associated volumes**.
5. Click **Remove**.

Result: containers, networks, AND volumes are permanently deleted.

## Step 3: Remove the Stack via CLI

```bash
# Remove stack without volumes (preserves data):
docker compose -p myapp down

# Remove stack AND all volumes (destructive!):
docker compose -p myapp down --volumes

# Remove stack, volumes, AND locally built images:
docker compose -p myapp down --volumes --rmi local

# Remove stack, volumes, ALL images (including pulled images):
docker compose -p myapp down --volumes --rmi all
```

## Step 4: Clean Up Orphaned Resources After Removal

Even after stack removal, some resources may persist:

```bash
# Check for orphaned volumes (not referenced by any container):
docker volume ls -f dangling=true

# Remove specific orphaned volumes:
docker volume rm myapp_postgres_data myapp_uploads

# Or prune all unused volumes:
docker volume prune --force   # WARNING: removes ALL unused volumes

# Check for orphaned networks:
docker network ls --filter type=custom
docker network prune --force  # Removes all custom networks with no containers

# Remove images from the stack (if no longer needed):
docker image rm myorg/api:latest myorg/web:latest
```

## Step 5: Remove a Stack That Won't Delete

If Portainer shows an error removing a stack:

```bash
# Check if containers are in a bad state:
docker ps -a --filter "label=com.docker.compose.project=myapp"

# Force remove stuck containers:
docker rm -f $(docker ps -aq --filter "label=com.docker.compose.project=myapp")

# Then remove the network manually:
docker network rm myapp_default myapp_frontend myapp_backend

# Remove the stack from Portainer (it will succeed now that containers are gone):
# Portainer UI → Stacks → Remove
```

## Step 6: Verify Complete Removal

Confirm the stack and its resources are fully gone:

```bash
# No containers from the stack:
docker ps -a --filter "label=com.docker.compose.project=myapp"
# Should return no rows

# Network is removed:
docker network ls | grep myapp
# Should return nothing

# Check if volumes still exist (expected if you didn't include --volumes):
docker volume ls | grep myapp
# myapp_postgres_data  ← This volume persists, which is intentional
```

## Step 7: Scheduled Stack Cleanup

For ephemeral stacks in CI/CD environments:

```bash
#!/bin/bash
# cleanup-old-stacks.sh - Remove stacks older than N days

# Via Portainer API:
curl -s "${PORTAINER_URL}/api/stacks" \
  -H "X-API-Key: ${PORTAINER_TOKEN}" | \
  jq -r '.[] | select(.Name | startswith("review-")) | .Id' | \
  while read stack_id; do
    echo "Removing stack: ${stack_id}"
    curl -X DELETE \
      "${PORTAINER_URL}/api/stacks/${stack_id}?endpointId=1" \
      -H "X-API-Key: ${PORTAINER_TOKEN}"
  done
```

## Conclusion

Removing a stack in Portainer is a two-decision process: remove now, and remove volumes or not. The default behavior preserves volumes to prevent accidental data loss - always back up important volumes before removal if you plan to delete them. Use the CLI (`docker compose down --volumes`) for scripted cleanup, and use Portainer's UI confirmation dialog for manual removals where you need to verify the volume decision. After removal, run `docker volume prune` and `docker network prune` to clean up any lingering orphaned resources.
