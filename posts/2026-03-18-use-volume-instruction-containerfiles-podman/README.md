# How to Use VOLUME Instruction in Containerfiles for Podman

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containerfile, Volume, Data Persistence, Storage

Description: Learn how to use the VOLUME instruction in Podman Containerfiles to manage persistent data, understand volume types, and implement data persistence patterns for containerized applications.

---

> The VOLUME instruction creates mount points for persistent data in your container images. Understanding how volumes work in Podman is essential for applications that need to store data beyond the container lifecycle.

Containers are ephemeral by design. When a container stops or is removed, all data written to the container's filesystem disappears. The VOLUME instruction in a Containerfile declares mount points where persistent or shared data should be stored, ensuring that important data survives container restarts and removals.

This guide covers the VOLUME instruction in Podman Containerfiles, explaining how volumes work, when to use the VOLUME instruction versus runtime volume mounts, and best practices for managing persistent data.

---

## Basic Syntax

The VOLUME instruction accepts one or more paths:

```dockerfile
# Single volume

VOLUME /data

# Multiple volumes
VOLUME /data /logs /config

# JSON array syntax
VOLUME ["/data", "/logs", "/config"]
```

## What VOLUME Does

The VOLUME instruction creates a mount point at the specified path. When a container is created from the image, Podman automatically creates an anonymous volume and mounts it at that path:

```dockerfile
FROM postgres:16-alpine

# PostgreSQL data directory is declared as a volume
VOLUME /var/lib/postgresql/data

EXPOSE 5432
```

Any data written to `/var/lib/postgresql/data` inside the container is stored in a Podman-managed volume rather than in the container's writable layer.

```bash
# Run the container - Podman creates an anonymous volume
podman run -d --name mydb postgres:16-alpine

# List volumes
podman volume ls
# Shows an auto-created volume with a random name

# Inspect to see the volume mount
podman inspect mydb --format '{{.Mounts}}'
```

## Understanding Volume Types in Podman

Podman supports several types of storage mounts:

### Named Volumes

Named volumes are managed by Podman and persist until explicitly removed:

```bash
# Create a named volume
podman volume create myapp-data

# Use it with a container
podman run -v myapp-data:/data myapp

# Inspect the volume
podman volume inspect myapp-data

# Remove the volume
podman volume rm myapp-data
```

### Bind Mounts

Bind mounts map a host directory into the container:

```bash
# Mount a host directory
podman run -v /host/path:/container/path myapp

# Read-only bind mount
podman run -v /host/config:/app/config:ro myapp

# Mount with SELinux labels (important on SELinux-enabled systems)
podman run -v /host/data:/data:Z myapp
```

### tmpfs Mounts

tmpfs mounts store data in memory, useful for sensitive or temporary data:

```bash
# Mount a tmpfs volume
podman run --mount type=tmpfs,destination=/tmp,tmpfs-size=100M myapp
```

### Anonymous Volumes

Created automatically by the VOLUME instruction or by `-v` without a name:

```bash
# Anonymous volume (auto-created by VOLUME instruction)
podman run myapp-with-volume

# Anonymous volume via CLI
podman run -v /data myapp
```

## VOLUME Instruction Behavior

Understanding the specific behavior of the VOLUME instruction helps you use it correctly.

### Data Initialization

When a named or anonymous volume is first mounted, Podman copies the existing data from the image into the volume:

```dockerfile
FROM alpine:3.19

# Create files in the image layer
WORKDIR /data
RUN echo "default config" > config.txt
RUN echo "initial data" > data.txt

# Declare as volume
VOLUME /data
```

The first time a container starts, `config.txt` and `data.txt` are copied into the volume. Subsequent containers using the same volume see the persisted data.

### Instructions After VOLUME

Any changes to a VOLUME path in subsequent Containerfile instructions are silently discarded:

```dockerfile
FROM alpine:3.19

VOLUME /data

# WARNING: This change will NOT be in the final image
RUN echo "important" > /data/file.txt
# The file is written to the anonymous volume during build,
# but a new volume is created when the container runs
```

Always make file changes before the VOLUME instruction:

```dockerfile
FROM alpine:3.19

# Create files BEFORE declaring the volume
RUN mkdir -p /data && echo "important" > /data/file.txt

VOLUME /data
```

## Practical Patterns

### Pattern 1: Database Container

```dockerfile
FROM postgres:16-alpine

ENV POSTGRES_DB=myapp
ENV POSTGRES_USER=appuser

# Data directory - persists database files
VOLUME /var/lib/postgresql/data

EXPOSE 5432
```

```bash
# Run with a named volume for data persistence
podman run -d \
    --name postgres \
    -v pgdata:/var/lib/postgresql/data \
    -e POSTGRES_PASSWORD=secret \
    mydb

# Data persists even after container removal
podman rm -f postgres
podman run -d \
    --name postgres-new \
    -v pgdata:/var/lib/postgresql/data \
    -e POSTGRES_PASSWORD=secret \
    mydb
# Database data is still there
```

### Pattern 2: Application with Logs and Uploads

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .

# Separate volumes for different data types
VOLUME /app/uploads
VOLUME /app/logs

USER 1001
EXPOSE 8000
CMD ["python", "app.py"]
```

```bash
podman run -d \
    -v app-uploads:/app/uploads \
    -v app-logs:/app/logs \
    -p 8000:8000 \
    myapp
```

### Pattern 3: Configuration Volume

```dockerfile
FROM nginx:alpine

# Copy default configuration
COPY nginx.conf /etc/nginx/nginx.conf
COPY default.conf /etc/nginx/conf.d/default.conf

# Static content served by nginx
COPY dist/ /usr/share/nginx/html/

# Volume for dynamic content
VOLUME /usr/share/nginx/html/uploads

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### Pattern 4: Cache Directory

```dockerfile
FROM node:20-alpine

WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .

# Cache directory as a volume
VOLUME /app/.cache

USER node
EXPOSE 3000
CMD ["node", "server.js"]
```

## Volume Permissions and Ownership

Volume permissions can be tricky, especially when running as a non-root user:

```dockerfile
FROM node:20-alpine

WORKDIR /app

# Create the volume directory with correct ownership BEFORE VOLUME
RUN mkdir -p /app/data && chown node:node /app/data

VOLUME /app/data

COPY --chown=node:node . .
RUN npm ci

USER node
CMD ["node", "server.js"]
```

For Podman in rootless mode, UID mapping adds complexity:

```bash
# Check UID mapping
podman unshare cat /proc/self/uid_map

# Run with user namespace mapping
podman run --userns=keep-id -v ./data:/app/data:Z myapp
```

## When to Use VOLUME vs Runtime Mounts

The VOLUME instruction in the Containerfile and the `-v` flag at runtime serve different purposes:

### Use VOLUME Instruction When:

- The image always needs persistent storage at that path (databases, file stores)
- You want to document which paths contain important data
- The path should never be part of the container's writable layer

```dockerfile
# Good use of VOLUME: Database always needs persistent storage
FROM postgres:16-alpine
VOLUME /var/lib/postgresql/data
```

### Use Runtime Mounts (-v) When:

- You want to control exactly where data is stored
- You need to share data between the host and container
- You want to provide configuration files
- The volume is optional or development-specific

```bash
# Development: Mount source code
podman run -v ./src:/app/src:Z myapp

# Production: Mount specific config
podman run -v /etc/myapp/config.yaml:/app/config.yaml:ro myapp
```

## Volume Management Commands

```bash
# Create a named volume
podman volume create myvolume

# List all volumes
podman volume ls

# Inspect a volume
podman volume inspect myvolume

# Remove a specific volume
podman volume rm myvolume

# Remove all unused volumes
podman volume prune

# Remove all unused volumes without confirmation prompt
podman volume prune --force

# Export volume data
podman volume export myvolume -o backup.tar

# Import volume data
podman volume import myvolume backup.tar
```

## Backup and Restore Patterns

```bash
# Backup a volume using a temporary container
podman run --rm \
    -v mydata:/source:ro \
    -v ./backups:/backup \
    alpine tar czf /backup/mydata-backup.tar.gz -C /source .

# Restore a volume from backup
podman run --rm \
    -v mydata:/target \
    -v ./backups:/backup:ro \
    alpine tar xzf /backup/mydata-backup.tar.gz -C /target
```

## SELinux Considerations

On SELinux-enabled systems (Fedora, RHEL, CentOS), bind mounts require proper labeling:

```bash
# :z - shared label (multiple containers can access)
podman run -v /host/data:/data:z myapp

# :Z - private label (only this container can access)
podman run -v /host/data:/data:Z myapp
```

Without these labels, the container may get "permission denied" errors when accessing mounted paths.

## Common Mistakes

```dockerfile
# Mistake 1: Modifying volume paths after VOLUME declaration
VOLUME /data
RUN echo "config" > /data/config.txt  # Silently discarded!

# Fix: Create files before VOLUME
RUN mkdir -p /data && echo "config" > /data/config.txt
VOLUME /data

# Mistake 2: Using VOLUME for temporary data
VOLUME /tmp  # Creates unnecessary persistent storage

# Mistake 3: Too many volumes
VOLUME /data /logs /cache /tmp /var /config  # Excessive

# Mistake 4: Not setting proper permissions
VOLUME /app/data
USER appuser
# appuser might not have write access to /app/data

# Fix: Set permissions before VOLUME
RUN mkdir -p /app/data && chown appuser:appuser /app/data
VOLUME /app/data
USER appuser
```

## Conclusion

The VOLUME instruction declares mount points for persistent data in your container images. It creates anonymous volumes automatically, initializes them with existing image data, and signals to users that the path contains important state. Use VOLUME for paths that genuinely need persistence, like database files and user uploads. Set permissions and create default content before the VOLUME declaration, as changes after it are discarded. For maximum control over storage, prefer named volumes and bind mounts at runtime with `podman run -v`. Understand the distinction between the VOLUME instruction as documentation and declaration versus runtime volume mounts as the operational mechanism for data persistence.
