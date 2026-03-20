# How to Run Portainer with Read-Only Root Filesystem - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Security, Read-Only, Container Hardening, Best Practices

Description: Harden containers by enabling read-only root filesystems, using tmpfs for writable directories, and applying immutable infrastructure principles via Portainer.

## Introduction

Running containers with a read-only root filesystem is one of the most effective container security hardening techniques. An attacker who gains code execution cannot modify system binaries, write persistence mechanisms, or install tools - because the filesystem is immutable. This guide covers enabling read-only root filesystems for containers in Portainer, handling directories that genuinely need writes, and validating the configuration.

## Step 1: Enable Read-Only Root Filesystem

```yaml
# docker-compose.yml - Read-only root filesystem

version: "3.8"

services:
  api:
    image: myapp/api:latest
    read_only: true  # Root filesystem is immutable

    # Applications often need to write to /tmp and /var/run
    tmpfs:
      - /tmp:size=100m,mode=1777       # Temporary files
      - /var/run:size=10m,mode=755      # PID files, sockets
      - /var/cache:size=50m             # Application cache

    volumes:
      # Data that must persist across restarts uses volumes
      - app_data:/app/data
      - app_logs:/app/logs

volumes:
  app_data:
  app_logs:
```

## Step 2: Via Docker CLI

```bash
# Run with read-only root
docker run -d \
  --read-only \
  --tmpfs /tmp:size=100m,mode=1777 \
  --tmpfs /var/run:size=10m \
  --name api \
  myapp/api:latest

# Test: attempt to write to root (should fail)
docker exec api touch /test_file
# Error: touch: /test_file: Read-only file system

# Confirm tmpfs is writable
docker exec api touch /tmp/test_file
# Succeeds - /tmp is tmpfs
```

## Step 3: Identify Required Write Locations

Before enabling read-only, audit what the application writes:

```bash
# Run normally first and trace filesystem writes
docker run -d --name api_audit myapp/api:latest

# Monitor filesystem activity for 30 seconds
docker run --rm --pid=container:api_audit \
  --privileged ubuntu:22.04 \
  bash -c "apt-get install -y strace && strace -p 1 -e trace=openat,open -f 2>&1 | grep 'O_WRONLY\|O_RDWR'"

# Or use inotifywait for simpler monitoring
docker exec api_audit bash -c "
  apt-get install -y inotify-tools
  inotifywait -r -m / --exclude /proc --exclude /sys 2>&1 | grep CREATE
"
```

## Step 4: Common Application Patterns

```yaml
# Node.js application
version: "3.8"

services:
  nodejs_app:
    image: node:18-alpine
    read_only: true
    tmpfs:
      - /tmp                    # npm, temp files
      - /app/node_modules/.cache  # Module cache
    volumes:
      - node_logs:/app/logs     # Persistent log volume
    working_dir: /app
    command: ["node", "server.js"]

  # Python/FastAPI application
  python_app:
    image: python:3.11-slim
    read_only: true
    tmpfs:
      - /tmp                    # pip temp, Python temp files
      - /root/.cache            # pip cache (if running as root)
    volumes:
      - python_data:/app/data
    command: ["uvicorn", "main:app", "--host", "0.0.0.0"]

  # Nginx
  nginx:
    image: nginx:alpine
    read_only: true
    tmpfs:
      - /var/cache/nginx        # Nginx cache
      - /var/run               # nginx.pid file
      - /tmp                   # Temp files
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro  # Config is read-only too
      - nginx_logs:/var/log/nginx

volumes:
  node_logs:
  python_data:
  nginx_logs:
```

## Step 5: Apply to Existing Portainer Stacks

In Portainer, update your stack's compose YAML to add `read_only: true` and the required `tmpfs` entries, then redeploy. Portainer will recreate the containers with the new security settings.

```bash
# Verify the container is running read-only
docker inspect api --format '{{.HostConfig.ReadonlyRootfs}}'
# Returns: true

# Check tmpfs mounts
docker inspect api --format '{{json .HostConfig.Tmpfs}}'
# Returns: {"/tmp":"size=100m,mode=1777","/var/run":"size=10m"}

# Confirm the container is still running (didn't crash due to write errors)
docker ps | grep api
```

## Step 6: Portainer Itself with Read-Only Filesystem

```yaml
# Hardened Portainer deployment
version: "3.8"

services:
  portainer:
    image: portainer/portainer-ce:latest
    read_only: true
    tmpfs:
      - /tmp
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - portainer_data:/data  # Portainer needs this for its database
    ports:
      - "9443:9443"
    restart: unless-stopped

volumes:
  portainer_data:
```

## Conclusion

Read-only root filesystems are a defense-in-depth security control that limits what an attacker can do after gaining container access. The key to success is identifying which directories genuinely need writes - usually `/tmp`, `/var/run`, and application-specific log/data directories - and handling them with tmpfs or named volumes. Start by testing with `--read-only` manually to identify write failures, then add the appropriate tmpfs mounts. Portainer shows the security configuration of each container in its detail view, making it easy to audit which containers have read-only protection enabled.
