# How to Fix Slow Stack Deployments Due to Registry Authentication

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Troubleshooting, Performance, Registry, Authentication, Docker Pull

Description: Learn how to fix slow stack deployments in Portainer caused by registry authentication delays, including credential caching, pull policy settings, and local registry mirrors.

---

Stack deployments that take minutes instead of seconds are often blocked on image pulls with slow registry authentication. This guide identifies the bottleneck and provides fixes.

## Step 1: Identify if the Delay is in Authentication

```bash
# Manually time an image pull to isolate the issue
time docker pull <image-name>:<tag>

# If the first few seconds are silent before pulling starts,
# the delay is in authentication (token fetch)
```

## Step 2: Test Registry Latency

```bash
# Test authentication endpoint latency
time curl -s -o /dev/null -w "%{time_total}" \
  https://auth.docker.io/token?service=registry.docker.io&scope=repository:library/ubuntu:pull

# For private registries
time curl -s -o /dev/null -w "%{time_total}" \
  https://your-registry.example.com/v2/
```

## Step 3: Pre-pull Images Before Deployment

For frequently deployed images, pre-pull them so Docker uses the local cache:

```bash
# Create a script to pre-pull all images used by stacks
docker pull myapp:1.2.3
docker pull postgres:16-alpine
docker pull redis:7-alpine

# Then deploy the stack - it will use local images and skip the pull
```

## Step 4: Set Pull Policy to Never in Portainer

For development environments where images do not change often:

```yaml
version: "3.8"
services:
  app:
    image: myapp:latest
    # Skip pull if image already exists locally
    pull_policy: if_not_present
```

## Step 5: Set Up a Local Registry Mirror

A local registry mirror caches Docker Hub images on your network:

```yaml
# Deploy a registry mirror
version: "3.8"
services:
  registry-mirror:
    image: registry:2
    ports:
      - "5000:5000"
    environment:
      REGISTRY_PROXY_REMOTEURL: https://registry-1.docker.io
    volumes:
      - registry_mirror:/var/lib/registry

volumes:
  registry_mirror:
```

Configure Docker to use the mirror in `/etc/docker/daemon.json`:

```json
{
  "registry-mirrors": ["http://localhost:5000"]
}
```

## Step 6: Cache Registry Credentials

For private registries, ensure credentials are cached to avoid re-authentication on every pull:

```bash
# Log in once to cache the token
docker login registry.example.com

# Docker stores credentials in ~/.docker/config.json
# This cache persists across container restarts
```

In Portainer, registries added under **Registries** are automatically used for pulls without requiring re-authentication at deployment time.
