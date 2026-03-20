# How to Remove Docker Networks in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Networking, Cleanup, DevOps

Description: Learn how to safely remove Docker networks in Portainer, including disconnecting containers, removing unused networks, and bulk cleanup.

## Introduction

Over time, Docker environments accumulate unused networks from removed stacks, completed projects, and development iterations. These networks consume minimal resources but add clutter and can cause subnet conflicts. Portainer and the Docker CLI both provide straightforward tools for removing networks — either individually or in bulk via prune operations.

## Prerequisites

- Portainer installed with a connected Docker environment

## When You Cannot Remove a Network

Docker will refuse to remove a network if:
1. Active containers are connected to it (running or stopped)
2. It is one of Docker's built-in networks: `bridge`, `host`, `none`

You must disconnect all containers before removal.

## Step 1: Identify Unused Networks via Portainer

1. Navigate to **Networks** in Portainer.
2. Look at the **Containers** column — networks with 0 containers are candidates for removal.
3. Check the **Scope** — `local` networks are per-host, `swarm` networks are cluster-wide.

## Step 2: Identify Unused Networks via CLI

```bash
# List all networks:
docker network ls

# Show networks with no containers:
docker network ls --filter "type=custom" --format "{{.ID}} {{.Name}}" | \
  while read id name; do
    count=$(docker network inspect "$id" --format '{{len .Containers}}')
    if [ "$count" -eq 0 ]; then
      echo "UNUSED: $name ($id)"
    fi
  done

# Quick filter for dangling networks:
docker network prune --dry-run 2>/dev/null || \
  echo "Note: dry-run not available, use 'docker network ls' to check"
```

## Step 3: Disconnect Containers from a Network

Before removing a network, disconnect all attached containers:

```bash
# List containers on a network:
docker network inspect my-old-network | jq '.[].Containers | to_entries[] | .value.Name'

# Disconnect each container:
docker network disconnect my-old-network container1
docker network disconnect my-old-network container2

# Force disconnect (use if container is stopped/unresponsive):
docker network disconnect --force my-old-network container1
```

Via Portainer:
1. Navigate to **Networks** → click the network name.
2. Under **Containers in this network**, click **Leave** for each container.

## Step 4: Remove a Single Network via Portainer

1. Navigate to **Networks** in Portainer.
2. Check the checkbox next to the network to remove.
3. Click **Remove** (or the trash icon).
4. Confirm deletion.

## Step 5: Remove a Single Network via CLI

```bash
# Remove by name:
docker network rm my-old-network

# Remove by ID:
docker network rm a1b2c3d4e5f6

# Remove multiple at once:
docker network rm network1 network2 network3

# Expected output:
# my-old-network
```

## Step 6: Prune All Unused Networks

```bash
# Remove all custom networks with no containers:
docker network prune

# Skip the confirmation prompt:
docker network prune --force

# Prune networks older than 24 hours:
docker network prune --filter "until=24h" --force

# Output shows what was removed:
# Deleted Networks:
# old-project-frontend
# test-network-1
# dev-backend
# Total reclaimed space: 0B  (networks use no disk, but this confirms success)
```

Via Portainer:
1. Navigate to **Networks**.
2. Click **Clean up unused networks** or **Prune**.
3. Confirm.

## Step 7: Remove Networks Created by Docker Compose

When a Compose stack is removed, its networks should be auto-removed. If they persist:

```bash
# Check if a compose-created network has containers from a stopped stack:
docker network inspect myapp_default

# Remove compose networks manually:
docker network rm myapp_frontend myapp_backend myapp_default

# Or use docker compose down which removes networks automatically:
cd /path/to/compose/project
docker compose down
# This removes containers AND networks (but not volumes unless --volumes is passed)
```

## Step 8: Handle Networks That Refuse to Delete

```bash
# Error: "active endpoints prevent network removal"
# Find all containers (including stopped ones) on the network:
docker ps -a --filter "network=stuck-network" --format "{{.Names}}"

# Remove stopped containers first, then retry:
docker container prune --filter "network=stuck-network" --force
docker network rm stuck-network

# If network still won't delete, restart Docker:
sudo systemctl restart docker
docker network rm stuck-network
```

## Conclusion

Removing Docker networks is straightforward once all containers are disconnected. Use `docker network prune` for bulk cleanup of unused custom networks, and `docker network rm` for targeted removal of specific networks. Portainer's Networks page shows container counts at a glance, making it easy to identify unused networks for cleanup. Networks created by Docker Compose are automatically removed by `docker compose down` — manual removal is only needed when stacks were deleted without running the compose down command.
