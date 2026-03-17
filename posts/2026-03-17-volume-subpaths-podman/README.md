# How to Use Volume Subpaths in Podman

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, DevOps, Volumes, Subpaths, Storage

Description: Learn how to mount specific subdirectories from a Podman volume into containers using subpath options.

---

> Volume subpaths let you mount a specific subdirectory from a volume rather than the entire volume, providing fine-grained control over what data containers can access.

Subpath mounts are useful when a single volume contains data for multiple services or when you want to expose only a portion of a volume to a container. Podman supports subpath mounting through the `--mount` flag.

---

## Basic Subpath Mount

```bash
# Create and populate a volume with multiple directories
podman volume create shared-data
podman run --rm -v shared-data:/data docker.io/library/alpine:latest \
  sh -c "mkdir -p /data/config /data/logs /data/uploads && \
         echo 'app.conf' > /data/config/app.conf && \
         echo 'log entry' > /data/logs/app.log"

# Mount only the config subdirectory
podman run --rm \
  --mount type=volume,source=shared-data,target=/app/config,subpath=config \
  docker.io/library/alpine:latest ls /app/config
# Output: app.conf
```

## Mounting Different Subpaths to Different Containers

```bash
# Config reader gets only the config subpath
podman run -d --name config-service \
  --mount type=volume,source=shared-data,target=/config,subpath=config \
  docker.io/library/alpine:latest tail -f /dev/null

# Log processor gets only the logs subpath
podman run -d --name log-processor \
  --mount type=volume,source=shared-data,target=/logs,subpath=logs \
  docker.io/library/alpine:latest tail -f /dev/null

# Upload handler gets only the uploads subpath
podman run -d --name upload-handler \
  --mount type=volume,source=shared-data,target=/uploads,subpath=uploads \
  docker.io/library/alpine:latest tail -f /dev/null
```

## Subpath with Read-Only Option

```bash
# Mount a subpath as read-only
podman run -d --name reader \
  --mount type=volume,source=shared-data,target=/app/config,subpath=config,readonly \
  docker.io/library/nginx:latest

# Verify read-only access
podman exec reader touch /app/config/test
# Output: touch: cannot touch '/app/config/test': Read-only file system
```

## Subpath for Configuration Isolation

```bash
# Volume structure:
# myapp-vol/
#   ├── nginx/
#   │   └── nginx.conf
#   ├── app/
#   │   └── settings.json
#   └── db/
#       └── my.cnf

# Each service gets only its relevant configuration
podman run -d --name web \
  --mount type=volume,source=myapp-vol,target=/etc/nginx/conf.d,subpath=nginx \
  docker.io/library/nginx:latest

podman run -d --name api \
  --mount type=volume,source=myapp-vol,target=/app/config,subpath=app \
  docker.io/library/node:20 tail -f /dev/null

podman run -d --name db \
  --mount type=volume,source=myapp-vol,target=/etc/mysql/conf.d,subpath=db \
  docker.io/library/mysql:8
```

## Subpath with Bind Mounts

```bash
# Bind mount subpath from a host directory
podman run -d --name app \
  --mount type=bind,source=/home/user/project,target=/app/src,bind-propagation=rprivate \
  docker.io/library/node:20 tail -f /dev/null
```

## Verifying Subpath Mounts

```bash
# Inspect the container mounts
podman inspect config-service --format '{{ json .Mounts }}'

# Verify only the subpath contents are visible
podman exec config-service ls -la /config
podman exec config-service ls /  # The parent volume dirs are not visible
```

## Summary

Volume subpaths in Podman allow you to mount specific subdirectories from a volume into containers, providing data isolation and access control. Use the `subpath` option with `--mount` to expose only the relevant portion of a shared volume to each container. Combine subpaths with `readonly` for additional security when containers only need to read configuration data.
