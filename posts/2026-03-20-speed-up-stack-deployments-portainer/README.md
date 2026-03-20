# How to Speed Up Stack Deployments in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Performance, Stack Deployment, CI/CD, Optimization

Description: Reduce stack deployment times in Portainer by pre-pulling images, using local registry mirrors, optimizing compose files, and leveraging Portainer's API for parallel deployments.

## Introduction

Slow stack deployments create friction in CI/CD pipelines and slow down incident response. The main bottlenecks are image pull time (downloading layers from remote registries), sequential container startup, and health check timeouts. This guide covers techniques to dramatically reduce deployment time for Portainer-managed stacks.

## Step 1: Pre-Pull Images Before Deployment

```bash
#!/bin/bash
# pre-pull.sh - Pull images before deployment to warm the cache

IMAGES=(
  "myapp/api:${BUILD_TAG}"
  "myapp/worker:${BUILD_TAG}"
  "nginx:alpine"
  "postgres:15-alpine"
  "redis:7-alpine"
)

echo "Pre-pulling images in parallel..."

# Pull all images concurrently
for image in "${IMAGES[@]}"; do
  docker pull "$image" &
done

# Wait for all pulls to complete
wait

echo "All images pre-pulled. Starting deployment..."

# Now trigger Portainer deployment (images already cached locally)
curl -s -X POST \
  -H "Authorization: Bearer $PORTAINER_TOKEN" \
  "https://portainer.example.com/api/webhooks/YOUR_WEBHOOK_ID"
```

## Step 2: Deploy a Local Registry Mirror

```yaml
# docker-compose.yml - Registry mirror for fast pulls
version: "3.8"

services:
  registry-mirror:
    image: registry:2
    container_name: registry_mirror
    restart: unless-stopped
    environment:
      - REGISTRY_PROXY_REMOTEURL=https://registry-1.docker.io
      - REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY=/data
      # Cache layers indefinitely (or set TTL)
      - REGISTRY_PROXY_TTL=168h
    volumes:
      - registry_mirror_data:/data
    ports:
      - "5000:5000"

volumes:
  registry_mirror_data:
```

```json
// /etc/docker/daemon.json - Point Docker at local mirror
{
  "registry-mirrors": ["http://registry-mirror.internal:5000"],
  "insecure-registries": ["registry-mirror.internal:5000"]
}
```

## Step 3: Optimize Docker Compose for Fast Startups

```yaml
# docker-compose.yml - Optimized for fast deployment
version: "3.8"

services:
  api:
    image: myapp/api:latest

    # Explicit dependency ordering prevents cascade failures
    depends_on:
      postgres:
        condition: service_healthy  # Wait for DB to be ready
      redis:
        condition: service_started  # Don't wait for full redis startup

    # Realistic health check (don't make it too slow)
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:8080/health"]
      interval: 5s      # Check frequently during startup
      timeout: 3s       # Quick timeout (don't wait too long)
      retries: 5        # Allow reasonable startup time
      start_period: 10s # Grace period before health checks count

    # Parallel startup: don't wait for previous container to finish
    # This is automatic in compose - all services start concurrently
    # unless depends_on creates a dependency

  postgres:
    image: postgres:15-alpine
    environment:
      - POSTGRES_DB=appdb
      - POSTGRES_USER=app
      - POSTGRES_PASSWORD=pass
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d appdb"]
      interval: 3s
      timeout: 5s
      retries: 10
      start_period: 20s

  redis:
    image: redis:7-alpine
    command: redis-server --loglevel warning  # Reduce startup logging
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 3s
      timeout: 3s
      retries: 5
```

## Step 4: Use Portainer API for Parallel Stack Updates

```bash
#!/bin/bash
# parallel-deploy.sh - Update multiple stacks simultaneously

PORTAINER_URL="https://portainer.example.com"
TOKEN="your_api_token"
ENDPOINT_ID=1

# Get stack IDs by name
get_stack_id() {
  local name=$1
  curl -s \
    -H "Authorization: Bearer $TOKEN" \
    "$PORTAINER_URL/api/stacks" | \
    jq -r ".[] | select(.Name == \"$name\") | .Id"
}

# Update a single stack
update_stack() {
  local stack_name=$1
  local compose_file=$2
  local stack_id=$(get_stack_id "$stack_name")

  echo "Updating stack: $stack_name (ID: $stack_id)"

  curl -s -X PUT \
    -H "Authorization: Bearer $TOKEN" \
    -H "Content-Type: application/json" \
    "$PORTAINER_URL/api/stacks/$stack_id?endpointId=$ENDPOINT_ID" \
    -d "{
      \"StackFileContent\": $(cat $compose_file | jq -Rs .),
      \"Prune\": false,
      \"PullImage\": true
    }"
}

# Deploy all stacks in parallel
update_stack "api-stack" "./stacks/api.yml" &
update_stack "worker-stack" "./stacks/worker.yml" &
update_stack "frontend-stack" "./stacks/frontend.yml" &

# Wait for all deployments
wait
echo "All stacks updated."
```

## Step 5: Layer Caching with Multi-Stage Builds

Smaller images pull faster. Optimize your Dockerfiles:

```dockerfile
# Dockerfile - Optimized layers for fast deployment

# Stage 1: Dependencies (changes rarely - cached layer)
FROM node:18-alpine AS dependencies
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production  # Cached unless package.json changes

# Stage 2: Build (changes with code)
FROM dependencies AS build
COPY . .
RUN npm run build

# Stage 3: Runtime (smallest possible image)
FROM node:18-alpine AS runtime
WORKDIR /app
# Only copy what's needed to run
COPY --from=dependencies /app/node_modules ./node_modules
COPY --from=build /app/dist ./dist

# Non-root user for security
USER node:node
EXPOSE 8080
CMD ["node", "dist/server.js"]
```

```bash
# Build with BuildKit cache mounts (faster repeated builds)
DOCKER_BUILDKIT=1 docker build \
  --cache-from registry.example.com/myapp/api:cache \
  --build-arg BUILDKIT_INLINE_CACHE=1 \
  -t registry.example.com/myapp/api:latest .

# Push cache layer
docker push registry.example.com/myapp/api:cache
```

## Step 6: Zero-Downtime Rolling Updates

```yaml
# docker-compose.yml with Swarm rolling update config
version: "3.8"

services:
  api:
    image: myapp/api:latest
    deploy:
      replicas: 3
      update_config:
        parallelism: 1          # Update 1 replica at a time
        delay: 5s               # Wait 5s between updates
        failure_action: rollback
        order: start-first      # Start new before stopping old
        monitor: 10s            # Monitor for 10s after update
      rollback_config:
        parallelism: 2
        delay: 0s
        order: start-first
```

```bash
# Measure actual deployment time
START=$(date +%s%N)

# Trigger deployment
curl -s -X POST \
  "https://portainer.example.com/api/webhooks/YOUR_ID"

# Wait for deployment to complete
docker service ls --filter "name=myapp_api" --format "{{.UpdateStatus.State}}" | \
  while read status; do
    [ "$status" = "completed" ] && break
    sleep 2
  done

END=$(date +%s%N)
ELAPSED=$(( (END - START) / 1000000 ))
echo "Deployment took: ${ELAPSED}ms"
```

## Conclusion

Deployment speed comes down to image availability and startup time. Pre-pulling images before triggering Portainer webhook deployments eliminates the pull wait. Local registry mirrors turn remote pulls into local cache hits. Optimized Dockerfiles with proper layer ordering reduce image sizes and maximize cache effectiveness. Well-tuned health checks with realistic `start_period` values prevent unnecessary retry cycles. Combining these techniques can reduce stack deployment time from several minutes to under 30 seconds in most environments.
