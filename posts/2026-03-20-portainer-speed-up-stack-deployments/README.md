# How to Speed Up Stack Deployments in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Stack Deployment, Performance, Docker, Registry, Image Pull

Description: Learn how to speed up Portainer stack deployments by pre-pulling images, configuring local registries, and optimizing stack files.

---

Slow stack deployments in Portainer are usually caused by slow image pulls. This guide covers pre-pulling images, configuring a local registry mirror, and structuring stacks for faster deployments.

## Why Deployments Are Slow

The main causes of slow deployments:

1. **Large images** - pulling a 2 GB image takes minutes on a slow connection
2. **No registry authentication cached** - re-authenticating on every pull
3. **Sequential image pulls** - images pulled one at a time
4. **No local cache** - same image pulled multiple times across nodes

## Solution 1: Pre-Pull Images Before Deployment

Pull images in advance so the `docker compose up` command uses the cache:

```bash
#!/bin/bash
# pre-pull-stack.sh docker-compose.yml

COMPOSE_FILE="${1:-docker-compose.yml}"

echo "Pre-pulling images from $COMPOSE_FILE..."
grep -E '^\s+image:' "$COMPOSE_FILE" | awk '{print $2}' | while read image; do
  echo "Pulling: $image"
  docker pull "$image"
done

echo "All images pre-pulled. Deploying stack..."
```

## Solution 2: Configure a Local Registry Mirror

A pull-through cache proxy eliminates repeat downloads. Deploy one via Portainer:

```yaml
version: "3.8"

services:
  registry-mirror:
    image: registry:2
    environment:
      REGISTRY_PROXY_REMOTEURL: https://registry-1.docker.io
      REGISTRY_PROXY_USERNAME: ""
      REGISTRY_PROXY_PASSWORD: ""
      REGISTRY_STORAGE_DELETE_ENABLED: "true"
    volumes:
      - registry_cache:/var/lib/registry
    ports:
      - "5001:5000"

volumes:
  registry_cache:
```

Configure Docker on each host to use the mirror in `/etc/docker/daemon.json`:

```json
{
  "registry-mirrors": ["http://registry-mirror-host:5001"]
}
```

Restart Docker after this change. Subsequent pulls for cached images serve locally at LAN speed.

## Solution 3: Use a Local Private Registry

Host your own registry for custom images to avoid public internet pulls:

```yaml
services:
  registry:
    image: registry:2
    environment:
      REGISTRY_HTTP_SECRET: mysecret
    volumes:
      - registry_data:/var/lib/registry
    ports:
      - "5000:5000"
```

Push your images to the local registry:

```bash
docker tag my-app:latest localhost:5000/my-app:latest
docker push localhost:5000/my-app:latest
```

Update stacks to pull from the local registry:

```yaml
services:
  api:
    image: localhost:5000/my-app:latest
```

## Solution 4: Multi-Stage Build Optimization

Reduce image sizes with multi-stage builds to minimize pull time:

```dockerfile
# Build stage - large dependencies

FROM node:20 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

# Runtime stage - minimal image
FROM node:20-alpine AS runtime
WORKDIR /app
COPY --from=builder /app/node_modules ./node_modules
COPY . .
EXPOSE 3000
CMD ["node", "server.js"]
```

A well-optimized multi-stage build reduces a 1.2 GB Node.js image to ~200 MB.

## Solution 5: Parallel Image Pulls

Pull all images simultaneously before deploying:

```bash
# Pull all images in parallel using background jobs
grep -E '^\s+image:' docker-compose.yml | awk '{print $2}' | \
  xargs -P 10 -I {} docker pull {}
echo "All pulls complete"
```

## Measuring Deployment Time

Benchmark your deployment to know which optimizations help most:

```bash
time docker compose -f docker-compose.yml up -d

# With pre-pulled images:
docker compose pull && time docker compose up -d
```

## Portainer Stack Deployment Timeout

If large stacks time out during Portainer deployment, the images are too large or the registry is too slow - solutions 2 and 3 above solve this. The Portainer deploy timeout is not user-configurable but pre-pulling images eliminates the risk.
