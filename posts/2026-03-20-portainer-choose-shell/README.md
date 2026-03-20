# How to Choose the Right Shell for Container Console in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Containers, Debugging, Linux

Description: Learn how to select the correct shell when opening a container console in Portainer based on the container's base image and available shells.

## Introduction

When opening a container console in Portainer, you must specify which shell to use. Choosing the wrong shell results in a "no such file or directory" error. The right shell depends on the base image — Alpine uses `/bin/sh`, Ubuntu/Debian have `/bin/bash`, and some minimal images have no shell at all.

## Prerequisites

- Portainer installed with a connected Docker environment
- A running container

## Step 1: Identify Available Shells

Before choosing a shell, determine which shells are available in the container:

```bash
# From Docker CLI (on the host), check available shells:
docker exec my-container ls -la /bin/sh /bin/bash /bin/ash /bin/zsh 2>/dev/null

# Or check /etc/shells:
docker exec my-container cat /etc/shells 2>/dev/null || echo "/etc/shells not found"
```

## Step 2: Shell Guide by Base Image

### Alpine Linux (`alpine:*`, `node:*-alpine`, `python:*-alpine`, etc.)

```
Available:   /bin/sh (BusyBox ash)
NOT available: /bin/bash
```

Use `/bin/sh` — not bash:

```bash
# In Portainer console dialog:
Shell: /bin/sh

# Common Alpine-based images:
# alpine:3.18
# nginx:alpine
# redis:7-alpine
# python:3.12-alpine
# node:20-alpine
# golang:1.22-alpine
```

Alpine's shell is actually BusyBox `ash`, a minimal POSIX shell. Most bash scripts work, but advanced bash features won't.

### Ubuntu/Debian (`ubuntu:*`, `debian:*`, most full images)

```
Available:   /bin/bash (full bash), /bin/sh (dash), /bin/dash
```

Use `/bin/bash` for full functionality:

```bash
Shell: /bin/bash

# Common Debian/Ubuntu-based images:
# ubuntu:22.04
# debian:12
# python:3.12  (Debian slim)
# node:20  (Debian slim)
# postgres:15  (Debian)
```

### Distroless Images

```
Available:   NONE (no shell at all)

# Images:
# gcr.io/distroless/java
# gcr.io/distroless/base
# gcr.io/distroless/python3
# chainguard/node
```

You cannot open a console in distroless containers. See the workaround section below.

### Official App Images (Nginx, Redis, etc.)

```bash
# Nginx (Alpine):
nginx:alpine → /bin/sh

# Nginx (Debian):
nginx:latest → /bin/bash

# Redis:
redis:7-alpine → /bin/sh
redis:7 → /bin/bash

# PostgreSQL:
postgres:15-alpine → /bin/sh
postgres:15 → /bin/bash
```

### Other Shells

Some images include additional shells:

```
zsh:       /bin/zsh (rare in containers, common in developer images)
fish:      /usr/bin/fish (developer tool images)
dash:      /bin/dash (Debian/Ubuntu's default /bin/sh is dash)
ksh:       /bin/ksh (some enterprise Linux images)
```

## Step 3: Test Shell Availability Before Console

Check from outside the container:

```bash
# Test which shells are available:
for shell in /bin/bash /bin/sh /bin/ash /bin/zsh /bin/dash; do
    if docker exec my-container test -f "$shell" 2>/dev/null; then
        echo "✓ ${shell} is available"
    else
        echo "✗ ${shell} not found"
    fi
done
```

## Step 4: Common Shell Selection Errors

### "No such file or directory: /bin/bash"

```bash
# Error: you specified /bin/bash but the image uses Alpine
# Fix: use /bin/sh instead

# In Portainer console dialog:
# Change from: /bin/bash
# Change to:   /bin/sh
```

### "exec: 'bash': executable file not found in $PATH"

Same issue — the image doesn't have bash. Switch to `/bin/sh`.

### "rpc error: code = 2 desc = cannot find executable"

The container might be distroless or the path is wrong. Try:

```bash
# Find any available shell:
docker exec my-container find / -name "sh" -o -name "bash" 2>/dev/null | head -5
```

## Step 5: Working with Minimal Shells

Alpine's `/bin/sh` (ash) supports most common commands but has limitations:

```bash
# Works in ash:
ls -la
cat file.txt
grep "pattern" file
ps aux
env
ping -c 4 host
wget -O- http://url
netstat -tlnp

# Does NOT work in ash (bash-specific features):
source /etc/profile.d/myscript.sh  # Use: . /etc/profile.d/myscript.sh
[[ -z "$VAR" ]]                    # Use: [ -z "$VAR" ]
echo {1..10}                       # Brace expansion not available
declare -A myarray                 # Associative arrays not available
```

## Step 6: Workaround for Distroless Images

When a container has no shell, use a debug sidecar or override approach:

### Method 1: Docker Debug (Docker Desktop feature)

```bash
# Docker Desktop: attach a debug shell to any container
docker debug my-distroless-app
```

### Method 2: Ephemeral Debug Container (Kubernetes approach)

In Kubernetes, use ephemeral containers. For standalone Docker, use:

```bash
# Run a shell in the same namespace as the target container
docker run -it --rm \
  --pid=container:my-distroless-app \
  --network=container:my-distroless-app \
  --volumes-from=my-distroless-app \
  busybox \
  /bin/sh
```

### Method 3: Debug Image Variant

Many projects provide debug image variants:

```dockerfile
# Production:
FROM gcr.io/distroless/java:17

# Debug variant (includes a shell):
FROM gcr.io/distroless/java:17-debug
```

## Step 7: Quick Reference

| Image Type | Shell to Use |
|-----------|-------------|
| Alpine (`*-alpine`) | `/bin/sh` |
| Ubuntu/Debian | `/bin/bash` |
| CentOS/RHEL | `/bin/bash` |
| BusyBox | `/bin/sh` |
| Distroless | Not available |
| Scratch | Not available |
| Most official images (non-alpine) | `/bin/bash` |

## Conclusion

Choosing the right shell in Portainer's console dialog is a matter of knowing your base image. Alpine and minimal images use `/bin/sh`, while full distro images (Ubuntu, Debian, CentOS) offer `/bin/bash`. For distroless images, use Docker debug containers or alternate debugging strategies. When in doubt, try `/bin/sh` first — it's more universally available than `/bin/bash`.
