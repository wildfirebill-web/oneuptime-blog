# How to Configure Docker Buildx with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Docker, IPv6, Buildx, BuildKit, Multi-Platform, Build

Description: Configure Docker Buildx and BuildKit to build container images in environments with IPv6 connectivity, handle IPv6 network access during build steps, and set up multi-platform builders with IPv6 support.

## Introduction

Docker Buildx uses BuildKit for image builds, which runs build steps in containers. When builds need network access (e.g., `apt-get install`, `npm install`), those containers need IPv6 connectivity if the packages are resolved via IPv6 addresses. Configuring BuildKit with proper network settings ensures build-time network operations work correctly in IPv6 and dual-stack environments.

## Create Buildx Builder with IPv6 Network

```bash
# Default builder uses host networking for build containers
# This inherits the host's IPv6 configuration

# Create a custom builder
docker buildx create \
    --name mybuilder \
    --driver docker-container \
    --driver-opt network=host \
    --use

# Verify builder is using host networking
docker buildx inspect mybuilder

# Build with IPv6 access (host networking during build)
docker buildx build \
    --builder mybuilder \
    --network host \
    -t myapp:latest .
```

## Build with IPv6 Network Access

```dockerfile
# Dockerfile — IPv6-aware build steps

FROM ubuntu:22.04

# Build with --network=host to inherit host IPv6
# Or ensure the build network has IPv6

# Install packages (may resolve over IPv6)
RUN apt-get update && apt-get install -y \
    curl \
    wget \
    && rm -rf /var/lib/apt/lists/*

# Download from IPv6-reachable server
RUN curl -6 -o /tmp/config.json https://config.example.com/config.json || \
    curl -4 -o /tmp/config.json https://config.example.com/config.json

# Test IPv6 connectivity during build
RUN curl -6 -s https://ipv6.icanhazip.com > /dev/null && \
    echo "IPv6 available in build" || \
    echo "IPv6 not available in build"
```

```bash
# Build with host network access (enables IPv6)
docker buildx build \
    --network=host \
    -t myapp:latest .

# Alternatively, build with a specific IPv6-enabled network
docker network create \
    --driver bridge \
    --ipv6 \
    --subnet fd00:build::/64 \
    build-net

docker buildx build \
    --network=build-net \
    -t myapp:latest .
```

## Multi-Platform Build with IPv6

```bash
# Create multi-platform builder
docker buildx create \
    --name multiplatform \
    --driver docker-container \
    --platform linux/amd64,linux/arm64 \
    --driver-opt network=host \
    --use

# Build for multiple platforms
docker buildx build \
    --platform linux/amd64,linux/arm64 \
    --network host \
    -t registry.example.com/myapp:latest \
    --push .

# Inspect built manifest
docker buildx imagetools inspect registry.example.com/myapp:latest
```

## BuildKit Configuration for IPv6

```toml
# /etc/buildkit/buildkitd.toml — BuildKit daemon configuration

[grpc]
  address = ["unix:///run/buildkit/buildkitd.sock"]

[worker.oci]
  # Allow builds to use IPv6
  networkMode = "host"

[registry."registry.example.com"]
  http = false
  insecure = false
```

```bash
# Start BuildKit daemon with custom config
buildkitd --config /etc/buildkit/buildkitd.toml &

# Or configure via Buildx driver options
docker buildx create \
    --name ipv6builder \
    --driver docker-container \
    --driver-opt "env.BUILDKIT_NETWORK=host" \
    --use
```

## Cache and IPv6 Registry

```bash
# Use IPv6 registry as build cache
docker buildx build \
    --cache-from type=registry,ref=[2001:db8::1]:5000/cache:myapp \
    --cache-to type=registry,ref=[2001:db8::1]:5000/cache:myapp,mode=max \
    -t myapp:latest .

# Export build cache to local directory
docker buildx build \
    --cache-to type=local,dest=/tmp/buildcache \
    -t myapp:latest .

# Use local cache in next build
docker buildx build \
    --cache-from type=local,src=/tmp/buildcache \
    -t myapp:latest .
```

## Conclusion

Configure Docker Buildx for IPv6 by using `--network=host` on both the builder creation and individual builds, which gives build containers access to the host's IPv6 network. Create a custom builder with `docker buildx create --driver-opt network=host` for persistent IPv6 access across builds. For Dockerfiles with build-time network operations, pass `--network=host` to `docker buildx build`. When using IPv6 registries for cache or push, use bracket notation `[ipv6]:port/image` in cache configuration. Multi-platform builds support the same IPv6 network options.
