# How to Manage Volumes with Podman Desktop

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, DevOps, Podman Desktop, Volumes, Storage

Description: Learn how to create, inspect, and manage persistent volumes in Podman Desktop to ensure your container data survives restarts and removals.

---

> Volumes decouple your data from container lifecycles, ensuring databases, uploads, and configuration persist across container restarts.

Containers are ephemeral by design. When a container is removed, its filesystem goes with it. Volumes solve this by providing persistent storage that exists independently of any container. Podman Desktop gives you a visual interface to create and manage volumes, while the CLI offers full control for scripting and automation.

---

## Understanding Podman Volumes

Volumes in Podman are managed storage locations on the host that can be mounted into containers. They are the preferred way to persist data because Podman handles their lifecycle, permissions, and cleanup.

```bash
# List all existing volumes

podman volume ls

# Get help on volume commands
podman volume --help
```

## Creating Volumes in Podman Desktop

To create a volume using the Podman Desktop UI:

1. Open Podman Desktop and click **Volumes** in the left sidebar.
2. Click the **Create Volume** button.
3. Enter a name for the volume (e.g., `db-data`).
4. Optionally configure labels for organization.
5. Click **Create** to finalize.

The volume will appear in the Volumes list, showing its name, creation date, and which containers are using it.

## Creating Volumes via the CLI

```bash
# Create a named volume
podman volume create db-data

# Create a volume with custom labels
podman volume create --label project=myapp --label env=dev app-uploads

# List volumes to confirm creation
podman volume ls

# Inspect a volume for mount point details
podman volume inspect db-data
```

The `inspect` command reveals the actual path on the host where data is stored, which is useful for debugging.

## Mounting Volumes to Containers

Once a volume exists, mount it into a container at startup:

```bash
# Run a PostgreSQL container with a persistent volume
podman run -d \
  --name my-postgres \
  -e POSTGRES_PASSWORD=mysecretpassword \
  -e POSTGRES_DB=myapp \
  -v db-data:/var/lib/postgresql/data \
  -p 5432:5432 \
  postgres:16

# Verify the volume is mounted
podman inspect my-postgres --format '{{.Mounts}}'
```

## Using Bind Mounts for Development

For development workflows, bind mounts map a host directory directly into the container:

```bash
# Mount your local source code into a Node.js container
podman run -d \
  --name dev-server \
  -v /Users/nawazdhandala/Projects/myapp/src:/app/src:Z \
  -p 3000:3000 \
  node:18-alpine sh -c "cd /app && npm start"
```

The `:Z` suffix tells Podman to relabel the content for SELinux compatibility, which is important on Linux hosts.

## Sharing Volumes Between Containers

Multiple containers can share the same volume for data exchange:

```bash
# Create a shared volume
podman volume create shared-data

# Container 1: writes data to the shared volume
podman run -d --name writer \
  -v shared-data:/data \
  alpine sh -c "while true; do date >> /data/log.txt; sleep 5; done"

# Container 2: reads from the same volume
podman run --rm \
  -v shared-data:/data:ro \
  alpine cat /data/log.txt
```

The `:ro` flag makes the mount read-only for the second container, preventing accidental writes.

## Backing Up and Restoring Volumes

Volumes can be backed up by copying their contents to an archive:

```bash
# Back up a volume to a tar file
podman run --rm \
  -v db-data:/source:ro \
  -v $(pwd):/backup \
  alpine tar czf /backup/db-data-backup.tar.gz -C /source .

# Restore a volume from a backup
podman volume create db-data-restored
podman run --rm \
  -v db-data-restored:/target \
  -v $(pwd):/backup:ro \
  alpine tar xzf /backup/db-data-backup.tar.gz -C /target
```

## Cleaning Up Volumes

Unused volumes consume disk space over time. Clean them up regularly:

```bash
# Remove a specific volume (must not be in use)
podman volume rm app-uploads

# Remove all unused volumes
podman volume prune -f

# Force remove a volume even if referenced
podman volume rm -f db-data

# Remove all volumes
podman volume rm --all
```

In Podman Desktop, you can select volumes and click **Delete** or use the **Prune** button to remove unused ones.

## Inspecting Volume Usage

To see which containers are using which volumes:

```bash
# List containers with their volume mounts
podman ps -a --format "{{.Names}}\t{{.Mounts}}"

# Get detailed volume information
podman volume inspect db-data --format '{{.Mountpoint}}'

# Check disk usage for volumes
podman system df -v
```

## Summary

Managing volumes in Podman Desktop ensures your container data persists across restarts and removals. Whether you use the graphical interface for quick operations or the CLI for automation, volumes are essential for databases, file uploads, and any stateful workload. Regular pruning keeps your system clean, and backup strategies protect against data loss.
