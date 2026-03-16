# How to Use Podman Farm for Multi-Architecture Builds

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, DevOps, Podman Farm, Multi-Architecture, Build Farm

Description: Learn the complete workflow for using Podman farm to build multi-architecture container images natively on different hardware, from setup to pushing images.

---

> Podman farm turns a collection of machines into a unified build system that produces native multi-arch images without emulation overhead.

QEMU emulation lets you build for any architecture on a single machine, but it is slow and sometimes unreliable for complex builds. Podman farm takes a different approach: it distributes builds to actual hardware running each target architecture. This guide walks through the end-to-end workflow.

---

## The Full Workflow

The multi-arch build workflow with Podman farm follows these steps:

1. Set up system connections to remote machines
2. Create a farm grouping those connections
3. Write an architecture-aware Containerfile
4. Build with `podman farm build`
5. Push the resulting manifest list to a registry

## Step 1: Set Up System Connections

```bash
# Add connections for each architecture
podman system connection add x86-builder \
    --identity ~/.ssh/podman_farm \
    ssh://builder@x86.build.example.com/run/user/1000/podman/podman.sock

podman system connection add arm-builder \
    --identity ~/.ssh/podman_farm \
    ssh://builder@arm.build.example.com/run/user/1000/podman/podman.sock

# Verify
podman system connection list
```

## Step 2: Create the Farm

```bash
# Group connections into a farm
podman farm create multiarch-farm x86-builder arm-builder

# Confirm
podman farm list
```

## Step 3: Write an Architecture-Aware Containerfile

```dockerfile
# Containerfile
FROM node:20-alpine AS builder

WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
COPY . .
RUN npm run build

FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY package.json .

EXPOSE 3000
USER node
CMD ["node", "dist/server.js"]
```

This Containerfile works on any architecture because the Node.js base images support multiple platforms. Each farm node pulls the correct base image for its architecture.

## Step 4: Build with the Farm

```bash
# Build across all farm nodes
podman farm build --farm multiarch-farm \
    -t registry.example.com/webapp:v2.0 \
    .
```

## Step 5: Push the Multi-Arch Manifest

```bash
# Log in to the registry
podman login registry.example.com

# Push the manifest list (includes all architectures)
podman manifest push --all \
    registry.example.com/webapp:v2.0 \
    docker://registry.example.com/webapp:v2.0
```

## Verifying the Result

```bash
# Inspect the manifest to see included platforms
podman manifest inspect registry.example.com/webapp:v2.0

# Or use skopeo for registry-side inspection
skopeo inspect --raw docker://registry.example.com/webapp:v2.0 | python3 -m json.tool
```

## Handling Native vs. Cross-Compiled Dependencies

Some applications have architecture-specific dependencies. Handle this in your Containerfile:

```dockerfile
FROM python:3.11-slim

# Architecture is handled natively by the farm node
# No need for cross-compilation flags
RUN pip install --no-cache-dir numpy pandas

COPY . /app
WORKDIR /app
CMD ["python", "main.py"]
```

Since each farm node builds natively, C extensions and native packages compile correctly without cross-compilation toolchains.

## Farm Build vs. QEMU Emulation Comparison

```bash
# QEMU approach (single machine, emulated)
time podman build --platform linux/arm64 -t myapp:arm64 .
# Typically 3-10x slower than native

# Farm approach (native hardware)
time podman farm build --farm multiarch-farm -t myapp:latest .
# Builds at native speed on each architecture
```

## Production CI/CD Integration

```bash
#!/bin/bash
# ci-farm-build.sh - Production multi-arch build script
set -euo pipefail

IMAGE="registry.example.com/myapp"
TAG="${CI_COMMIT_TAG:-${CI_COMMIT_SHA:0:8}}"
FARM="prod-farm"

echo "=== Farm Build: ${IMAGE}:${TAG} ==="

# Verify farm health
CONNECTIONS=$(podman farm list --format '{{if eq .Name "'"${FARM}"'"}}{{.Connections}}{{end}}' | tr ',' ' ')

HEALTHY=true
for CONN in ${CONNECTIONS}; do
    if ! podman --connection "${CONN}" info >/dev/null 2>&1; then
        echo "WARNING: Node ${CONN} is unreachable"
        HEALTHY=false
    fi
done

if [ "${HEALTHY}" = false ]; then
    echo "ERROR: Not all farm nodes are reachable. Aborting."
    exit 1
fi

# Build
podman farm build --farm "${FARM}" -t "${IMAGE}:${TAG}" .

# Push
podman login -u "${REGISTRY_USER}" -p "${REGISTRY_PASS}" registry.example.com
podman manifest push --all "${IMAGE}:${TAG}" "docker://${IMAGE}:${TAG}"

# Also tag as latest if this is a release tag
if [[ "${TAG}" =~ ^v[0-9] ]]; then
    podman manifest push --all "${IMAGE}:${TAG}" "docker://${IMAGE}:latest"
fi

echo "=== Build complete ==="
```

## Summary

Podman farm provides native multi-architecture builds by distributing work to real hardware over SSH. The workflow is straightforward: configure connections, create a farm, build with `podman farm build`, and push the resulting manifest list. Native builds avoid QEMU performance penalties and produce more reliable results for applications with architecture-specific dependencies.
