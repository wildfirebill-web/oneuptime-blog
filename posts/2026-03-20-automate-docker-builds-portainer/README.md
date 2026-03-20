# How to Automate Docker Image Builds and Deployments with Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Automation, CI/CD, Watchtower, Build Automation

Description: Automate Docker image builds with Buildx, push to a registry, and trigger Portainer deployments using webhooks and Watchtower.

## Introduction

Automating Docker builds and deployments eliminates manual steps and ensures your containers are always running the latest code. This guide covers setting up automated builds with Docker Buildx, pushing to registries, and triggering Portainer redeployments via webhooks and Watchtower.

## Step 1: Set Up a Private Registry with Auto-Build

```yaml
# docker-compose.yml - Private registry with build automation
version: "3.8"

networks:
  registry_net:
    driver: bridge

volumes:
  registry_data:
  buildkitd_data:

services:
  # Private Docker Registry
  registry:
    image: registry:2
    container_name: private_registry
    restart: unless-stopped
    ports:
      - "5000:5000"
    environment:
      - REGISTRY_HTTP_SECRET=registry_secret
      - REGISTRY_STORAGE_DELETE_ENABLED=true
      # Enable pull-through cache for Docker Hub
      - REGISTRY_PROXY_REMOTEURL=https://registry-1.docker.io
    volumes:
      - registry_data:/var/lib/registry
    networks:
      - registry_net

  # BuildKit daemon for remote builds
  buildkitd:
    image: moby/buildkit:buildx-stable-1
    container_name: buildkitd
    restart: unless-stopped
    privileged: true
    volumes:
      - buildkitd_data:/var/lib/buildkit
    networks:
      - registry_net
```

## Step 2: Automated Build Script

```bash
#!/bin/bash
# /usr/local/bin/build-and-deploy.sh
# Automated Docker build and Portainer deployment

set -euo pipefail

# Configuration
APP_NAME="${1:-myapp}"
REGISTRY="${REGISTRY:-registry.yourdomain.com}"
PORTAINER_URL="${PORTAINER_URL:-https://portainer.yourdomain.com}"
PORTAINER_API_KEY="${PORTAINER_API_KEY:?Required}"
STACK_ID="${STACK_ID:?Required}"

# Compute image tag
GIT_COMMIT=$(git rev-parse --short HEAD 2>/dev/null || echo "unknown")
BUILD_TIME=$(date +%Y%m%d%H%M%S)
IMAGE_TAG="${BUILD_TIME}-${GIT_COMMIT}"
IMAGE="${REGISTRY}/${APP_NAME}:${IMAGE_TAG}"
IMAGE_LATEST="${REGISTRY}/${APP_NAME}:latest"

log() { echo "[$(date '+%H:%M:%S')] $1"; }

# ==================
# Build with BuildKit
# ==================
build_image() {
    log "Building: $IMAGE"

    # Build with BuildKit (faster, layer caching)
    DOCKER_BUILDKIT=1 docker build \
        --build-arg BUILD_DATE="$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
        --build-arg GIT_COMMIT="$GIT_COMMIT" \
        --label "build.number=$BUILD_TIME" \
        --label "git.commit=$GIT_COMMIT" \
        -t "$IMAGE" \
        -t "$IMAGE_LATEST" \
        .

    log "Build complete"
}

# ==================
# Security Scan
# ==================
scan_image() {
    if command -v trivy &>/dev/null; then
        log "Scanning for vulnerabilities..."
        trivy image \
            --exit-code 1 \
            --severity HIGH,CRITICAL \
            --ignore-unfixed \
            "$IMAGE" || {
            log "WARNING: Vulnerabilities found"
            return 0  # Non-blocking for now
        }
    fi
}

# ==================
# Push to Registry
# ==================
push_image() {
    log "Pushing: $IMAGE"
    docker push "$IMAGE"
    docker push "$IMAGE_LATEST"
    log "Push complete"
}

# ==================
# Update Portainer
# ==================
deploy_to_portainer() {
    log "Deploying to Portainer stack $STACK_ID"

    RESPONSE=$(curl -s -w "\n%{http_code}" -X PUT \
        -H "X-API-Key: $PORTAINER_API_KEY" \
        -H "Content-Type: application/json" \
        "$PORTAINER_URL/api/stacks/$STACK_ID?endpointId=1" \
        -d "{
          \"env\": [
            {\"name\": \"IMAGE_TAG\", \"value\": \"$IMAGE_TAG\"},
            {\"name\": \"APP_VERSION\", \"value\": \"$GIT_COMMIT\"}
          ],
          \"prune\": false,
          \"pullImage\": true
        }")

    HTTP_STATUS=$(echo "$RESPONSE" | tail -1)
    BODY=$(echo "$RESPONSE" | head -1)

    if [ "$HTTP_STATUS" = "200" ]; then
        log "Deployment successful!"
        log "Stack updated with IMAGE_TAG=$IMAGE_TAG"
    else
        log "ERROR: Deployment failed (HTTP $HTTP_STATUS)"
        log "Response: $BODY"
        exit 1
    fi
}

# ==================
# Main
# ==================
main() {
    log "=== Starting automated build and deploy ==="
    build_image
    scan_image
    push_image
    deploy_to_portainer
    log "=== Complete: $IMAGE_TAG deployed ==="
}

main
```

## Step 3: Watchtower for Automatic Container Updates

Watchtower monitors running containers and automatically updates them when new images are available:

```yaml
# docker-compose.yml - Watchtower configuration
version: "3.8"

services:
  watchtower:
    image: containrrr/watchtower
    container_name: watchtower
    restart: unless-stopped
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      # Check for updates every 5 minutes
      - WATCHTOWER_SCHEDULE=*/5 * * * *
      # Remove old images after update
      - WATCHTOWER_CLEANUP=true
      # Only update containers with this label
      - WATCHTOWER_LABEL_ENABLE=true
      # Notify via email
      - WATCHTOWER_NOTIFICATIONS=email
      - WATCHTOWER_NOTIFICATION_EMAIL_FROM=watchtower@yourdomain.com
      - WATCHTOWER_NOTIFICATION_EMAIL_TO=admin@yourdomain.com
      - WATCHTOWER_NOTIFICATION_EMAIL_SERVER=smtp.gmail.com
      - WATCHTOWER_NOTIFICATION_EMAIL_SERVER_PORT=587
      - WATCHTOWER_NOTIFICATION_EMAIL_SERVER_USER=your-email@gmail.com
      - WATCHTOWER_NOTIFICATION_EMAIL_SERVER_PASSWORD=your-app-password
      # Include system start event in notifications
      - WATCHTOWER_NOTIFICATION_EMAIL_DELAY=2
      # HTTP API for triggering updates
      - WATCHTOWER_HTTP_API_TOKEN=watchtower_api_token
      - WATCHTOWER_HTTP_API_UPDATE=true
    ports:
      - "8080:8080"

  # Application with Watchtower label
  my_app:
    image: registry.yourdomain.com/myapp:latest
    container_name: my_app
    restart: unless-stopped
    labels:
      # Watchtower will auto-update this container
      - "com.centurylinklabs.watchtower.enable=true"
      # Or use specific image for comparison
      - "com.centurylinklabs.watchtower.scope=myapp-scope"
```

## Step 4: Trigger Watchtower via Webhook

```bash
# Trigger update check immediately after pushing new image
curl -H "Authorization: Bearer watchtower_api_token" \
     http://watchtower-host:8080/v1/update

# Trigger update for specific container
curl -H "Authorization: Bearer watchtower_api_token" \
     http://watchtower-host:8080/v1/update/my_app
```

## Step 5: Multi-Architecture Builds

```bash
# Build for multiple architectures (arm64 for Raspberry Pi + amd64 for servers)
docker buildx create --use --name multiarch_builder

docker buildx build \
    --platform linux/amd64,linux/arm64,linux/arm/v7 \
    --push \
    -t "${REGISTRY}/${APP_NAME}:${IMAGE_TAG}" \
    -t "${REGISTRY}/${APP_NAME}:latest" \
    .
```

## Step 6: Makefile for Build Automation

```makefile
# Makefile - Standardize build commands

REGISTRY ?= registry.yourdomain.com
APP_NAME ?= myapp
GIT_COMMIT := $(shell git rev-parse --short HEAD)
BUILD_TIME := $(shell date +%Y%m%d%H%M%S)
IMAGE_TAG := $(BUILD_TIME)-$(GIT_COMMIT)
IMAGE := $(REGISTRY)/$(APP_NAME):$(IMAGE_TAG)

.PHONY: build push deploy all

build:
	DOCKER_BUILDKIT=1 docker build -t $(IMAGE) -t $(REGISTRY)/$(APP_NAME):latest .

push: build
	docker push $(IMAGE)
	docker push $(REGISTRY)/$(APP_NAME):latest

test:
	docker run --rm -v "$(PWD):/app" -w /app python:3.12-slim \
		sh -c "pip install -q -r requirements-test.txt && pytest tests/"

deploy: push
	./build-and-deploy.sh $(APP_NAME)

all: test build push deploy
```

## Conclusion

Automated Docker builds with Portainer deployments create a reliable, repeatable deployment process. The build script handles the full lifecycle from build to scan to deploy, Watchtower provides hands-off updates for simpler services, and the Portainer API gives you fine-grained control when you need it. CI/CD pipelines using these patterns ensure every deployment is consistent, traceable, and reversible.
