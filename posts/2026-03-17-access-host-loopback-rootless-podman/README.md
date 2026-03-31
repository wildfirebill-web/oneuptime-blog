# How to Access Host Loopback from a Rootless Podman Container

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Container, DevOps, Networking, Rootless, Loopback, Development

Description: Learn how to access host localhost services from rootless Podman containers using different networking backends.

---

> Accessing host loopback (127.0.0.1) from rootless containers requires specific configuration depending on the networking backend in use.

In rootless Podman, containers run in a separate user and network namespace. Services listening on the host's 127.0.0.1 are not directly reachable from inside the container. This guide covers the methods for accessing host loopback services from rootless containers.

---

## The Loopback Access Problem

```bash
# A service running on host localhost

# Example: Node.js dev server on http://127.0.0.1:3000

# By default, rootless containers cannot reach host localhost
podman run --rm docker.io/library/alpine:latest \
  sh -c "apk add --no-cache curl > /dev/null 2>&1 && curl http://127.0.0.1:3000"
# curl: (7) Failed to connect
```

## Method 1: Using host.containers.internal

```bash
# The special hostname resolves to the host
podman run --rm docker.io/library/alpine:latest \
  ping -c 2 host.containers.internal

# Access a host service
podman run --rm docker.io/library/alpine:latest \
  sh -c "apk add --no-cache curl > /dev/null 2>&1 && curl http://host.containers.internal:3000"
```

## Method 2: Pasta with Loopback Access

Pasta provides the best loopback access for rootless containers:

```bash
# Pasta allows host loopback access by default
podman run --rm --network pasta \
  docker.io/library/alpine:latest \
  sh -c "apk add --no-cache curl > /dev/null 2>&1 && curl -s http://host.containers.internal:3000"

# Map gateway to host loopback
podman run --rm --network pasta:--map-gw \
  docker.io/library/alpine:latest \
  sh -c "apk add --no-cache curl > /dev/null 2>&1 && curl -s http://host.containers.internal:3000"
```

## Method 3: slirp4netns with allow_host_loopback

```bash
# Enable loopback access in slirp4netns
podman run --rm \
  --network slirp4netns:allow_host_loopback=true \
  docker.io/library/alpine:latest \
  sh -c "apk add --no-cache curl > /dev/null 2>&1 && curl -s http://10.0.2.2:3000"

# The host gateway is 10.0.2.2 in slirp4netns
```

## Method 4: Using --add-host

```bash
# Map a friendly hostname to the host gateway
podman run --rm \
  --add-host host:host-gateway \
  docker.io/library/alpine:latest \
  sh -c "apk add --no-cache curl > /dev/null 2>&1 && curl -s http://host:3000"
```

## Method 5: Host Network Mode

```bash
# Host networking gives direct loopback access
podman run --rm --network host \
  docker.io/library/alpine:latest \
  sh -c "apk add --no-cache curl > /dev/null 2>&1 && curl -s http://127.0.0.1:3000"

# But loses network isolation
```

## Development Workflow Example

```bash
# Start your backend on the host
# cd /home/user/api && npm start  (listening on 127.0.0.1:4000)

# Run your frontend container with host access
podman run -d --name frontend \
  --add-host api:host-gateway \
  -e API_URL=http://api:4000 \
  -p 3000:3000 \
  my-frontend:dev

# The frontend container can reach the host's API server
```

## Configuring Default Loopback Access

```bash
# Set in containers.conf for automatic loopback access
# Edit ~/.config/containers/containers.conf

# For slirp4netns:
# [containers]
# default_rootless_network_cmd = "slirp4netns"
#
# [engine]
# network_cmd_options = ["allow_host_loopback=true"]

# For pasta (default has loopback access):
# [containers]
# default_rootless_network_cmd = "pasta"
```

## Verifying Host Loopback Access

```bash
# Check which services are on host loopback
ss -tlnp | grep 127.0.0.1

# Test from the container
podman run --rm \
  --add-host host:host-gateway \
  docker.io/library/alpine:latest \
  sh -c "
    apk add --no-cache curl > /dev/null 2>&1
    echo 'Testing host loopback services:'
    curl -s -o /dev/null -w '%{http_code}' http://host:3000 && echo ' - Port 3000: OK' || echo ' - Port 3000: FAIL'
    curl -s -o /dev/null -w '%{http_code}' http://host:5432 && echo ' - Port 5432: OK' || echo ' - Port 5432: FAIL'
  "
```

## Summary

Access host loopback services from rootless Podman containers using `host.containers.internal`, pasta networking (which has loopback access by default), slirp4netns with `allow_host_loopback=true`, or `--add-host` with `host-gateway`. Pasta is the recommended approach for the easiest setup. For development workflows, map descriptive hostnames to `host-gateway` so application configuration stays clean and readable.
