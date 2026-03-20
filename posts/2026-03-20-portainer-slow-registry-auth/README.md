# How to Fix Slow Stack Deployments Due to Registry Authentication

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Performance, Registry, Troubleshooting, Stacks

Description: Speed up slow Portainer stack deployments caused by registry authentication timeouts, DNS resolution delays, and inefficient image pull strategies.

## Introduction

Stack deployments in Portainer can become extremely slow when registry authentication takes long — 30+ seconds or more for each image pull. This is commonly caused by DNS resolution delays for registry servers, stale authentication tokens, or pull-always policies on large images. This guide explains how to diagnose and fix each bottleneck.

## Step 1: Identify the Bottleneck

```bash
# Time a manual image pull to measure the actual bottleneck
time docker pull myregistry.com/myimage:v1.0

# Compare stages:
# - "Pulling from..." appearing quickly = auth is fast
# - Long pause before "Pulling from..." = DNS/auth is the bottleneck
# - "Pulling fs layer" is slow = network bandwidth issue
# - "Waiting" messages = rate limiting or concurrent pull throttling
```

## Step 2: Test DNS Resolution Speed

```bash
# Check how long DNS takes for your registry
time nslookup myregistry.com

# Check from inside Docker (may use different DNS)
docker run --rm busybox time nslookup myregistry.com

# If DNS is slow, configure a faster resolver
cat /etc/resolv.conf

# Set Docker to use a fast DNS (e.g., Cloudflare or Google)
cat > /etc/docker/daemon.json << 'EOF'
{
  "dns": ["1.1.1.1", "8.8.8.8"],
  "dns-search": []
}
EOF

sudo systemctl restart docker
```

## Step 3: Pre-authenticate to Registries

Authentication happens at pull time. Pre-authentication caches credentials:

```bash
# Pre-authenticate to Docker Hub
docker login

# Pre-authenticate to custom registry
docker login myregistry.com -u username -p password

# Pre-authenticate to ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  123456789.dkr.ecr.us-east-1.amazonaws.com

# Docker caches credentials in ~/.docker/config.json
cat ~/.docker/config.json
```

## Step 4: Configure Portainer Registry Credentials

Ensure Portainer has valid, non-expired credentials:

1. Go to **Registries** in Portainer
2. Edit each registry and re-enter credentials
3. For ECR: update the password (token) as they expire every 12 hours

```bash
# Automate ECR credential refresh
#!/bin/bash
# refresh-portainer-ecr.sh

PORTAINER_URL="http://localhost:9000"
ECR_REGISTRY="123456789.dkr.ecr.us-east-1.amazonaws.com"

# Get Portainer auth token
TOKEN=$(curl -s -X POST "$PORTAINER_URL/api/auth" \
  -H "Content-Type: application/json" \
  -d '{"Username":"admin","Password":"yourpassword"}' | jq -r .jwt)

# Get fresh ECR token
ECR_TOKEN=$(aws ecr get-login-password --region us-east-1)

# Find the ECR registry ID in Portainer
REGISTRY_ID=$(curl -s -H "Authorization: Bearer $TOKEN" \
  "$PORTAINER_URL/api/registries" | \
  jq -r ".[] | select(.URL | contains(\"$ECR_REGISTRY\")) | .Id")

# Update the registry password
curl -X PUT \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "$PORTAINER_URL/api/registries/$REGISTRY_ID" \
  -d "{\"Password\": \"$ECR_TOKEN\", \"Username\": \"AWS\"}"
```

## Step 5: Enable Local Image Cache

For frequently used images, pull them locally before deployment:

```bash
# Pre-pull images on the Docker host before deploying stacks
docker pull myregistry.com/myapp:v1.0
docker pull postgres:16
docker pull redis:7.2

# Now stack deployments use the local cache (near-instant)
```

Configure Portainer stacks to not always re-pull:
1. In stack deployment settings, uncheck **Re-pull image**
2. This uses the local cache if the image exists

## Step 6: Use a Local Registry Mirror

Set up a pull-through cache registry to mirror Docker Hub:

```bash
# Deploy a registry mirror
docker run -d \
  -p 6000:5000 \
  --name registry-mirror \
  --restart always \
  -e REGISTRY_PROXY_REMOTEURL=https://registry-1.docker.io \
  -v registry_mirror_data:/var/lib/registry \
  registry:2

# Configure Docker to use the mirror
cat > /etc/docker/daemon.json << 'EOF'
{
  "registry-mirrors": ["http://localhost:6000"]
}
EOF

sudo systemctl restart docker

# Now all Docker Hub pulls go through local cache
docker pull nginx:latest  # Uses local mirror cache
```

## Step 7: Optimize Stack Deploy for Multi-Image Stacks

For stacks with many services, pulls happen sequentially by default:

```bash
# Use docker compose pull --parallel for faster multi-image pulls
cd /opt/stacks/mystack
docker compose pull --parallel

# After pre-pulling, deploy in Portainer with Re-pull disabled
```

## Step 8: Configure Image Pull Parallelism in Docker

```bash
# Docker daemon can pull multiple images simultaneously
cat > /etc/docker/daemon.json << 'EOF'
{
  "max-concurrent-downloads": 10,
  "max-concurrent-uploads": 5
}
EOF

sudo systemctl restart docker
```

## Step 9: Use Image Digest Pinning

Pinned digests avoid the registry manifest lookup overhead:

```yaml
# Instead of tag-based references (require manifest lookup)
image: nginx:latest

# Use digest references (deterministic, faster auth)
image: nginx@sha256:abc123...definite_hash_here
```

```bash
# Get the digest for a specific tag
docker pull nginx:latest
docker inspect nginx:latest --format='{{index .RepoDigests 0}}'
```

## Step 10: Monitor Pull Times with Portainer Logs

```bash
# Monitor deployment timing in Portainer
docker logs portainer 2>&1 | grep -i "pull\|registry\|image" | tail -30

# Time the full stack deployment
time docker compose -f /opt/stacks/mystack/docker-compose.yml pull
time docker compose -f /opt/stacks/mystack/docker-compose.yml up -d
```

## Conclusion

Slow stack deployments due to registry authentication are most commonly caused by DNS resolution delays and expired or missing authentication tokens. The fastest fixes are configuring fast DNS resolvers in the Docker daemon, pre-pulling images to warm the local cache, and setting up a local registry mirror for frequently used base images. For private registries, ensure credentials are current and non-expired, especially for time-limited tokens like AWS ECR.
