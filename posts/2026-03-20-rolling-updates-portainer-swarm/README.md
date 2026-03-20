# How to Implement Rolling Updates with Portainer on Swarm

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker Swarm, Rolling Updates, Zero Downtime, DevOps, Deployment

Description: Configure Docker Swarm rolling updates with Portainer for zero-downtime deployments with automatic rollback on failure.

## Introduction

Rolling updates replace containers one at a time (or in small batches), ensuring your service stays available throughout the update. Docker Swarm has built-in rolling update support with configurable parallelism, delay, health checks, and rollback policies. Portainer provides a visual interface to manage and monitor these updates.

## Step 1: Deploy Service with Update Config

```yaml
# docker-compose.yml - Service with rolling update configuration
version: "3.8"

networks:
  app_overlay:
    driver: overlay
    attachable: true

services:
  api:
    image: myapp/api:${IMAGE_TAG:-latest}
    networks:
      - app_overlay
    deploy:
      replicas: 6

      # Rolling update configuration
      update_config:
        # Update 2 replicas at a time
        parallelism: 2
        # Wait 15s between updating batches
        delay: 15s
        # Start new replica before stopping old (zero downtime)
        order: start-first
        # Wait 60s before marking update as successful
        monitor: 60s
        # Roll back all replicas on failure
        failure_action: rollback
        # Maximum failure ratio before rollback
        max_failure_ratio: 0

      # Rollback configuration (when failure_action=rollback)
      rollback_config:
        parallelism: 2
        delay: 5s
        order: stop-first
        failure_action: pause
        monitor: 30s

      # Restart policy for failed containers
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
        window: 60s

      # Resource limits
      resources:
        limits:
          cpus: "0.5"
          memory: 256M

    # Health check ensures container is ready before update proceeds
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 30s

    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.api.rule=Host(`api.yourdomain.com`)"
      - "traefik.http.services.api.loadbalancer.server.port=8000"
      - "traefik.http.services.api.loadbalancer.healthcheck.path=/health"
      - "traefik.http.services.api.loadbalancer.healthcheck.interval=10s"
```

## Step 2: Trigger Rolling Update via Portainer

### Via Portainer UI
1. Navigate to **Services** in Portainer (Swarm mode)
2. Click the **api** service
3. Click **Update** or **Edit**
4. Change the image tag
5. Click **Update the service**

### Via Portainer API

```bash
#!/bin/bash
# rolling-update.sh - Trigger rolling update via Portainer API

NEW_IMAGE="myapp/api:${1:?Specify new image tag}"
SERVICE_ID="myapp_api"
PORTAINER_URL="https://portainer.yourdomain.com"
API_KEY="your-portainer-api-key"
ENDPOINT_ID=1

echo "Starting rolling update: $SERVICE_ID → $NEW_IMAGE"

# Get current service spec
SERVICE=$(curl -s \
    -H "X-API-Key: $API_KEY" \
    "$PORTAINER_URL/api/endpoints/$ENDPOINT_ID/docker/services/$SERVICE_ID")

CURRENT_VERSION=$(echo "$SERVICE" | jq -r '.Version.Index')

echo "Service version: $CURRENT_VERSION"

# Update the service with new image
UPDATE_RESPONSE=$(curl -s -X POST \
    -H "X-API-Key: $API_KEY" \
    -H "Content-Type: application/json" \
    "$PORTAINER_URL/api/endpoints/$ENDPOINT_ID/docker/services/$SERVICE_ID/update?version=$CURRENT_VERSION" \
    -d "$(echo "$SERVICE" | jq --arg img "$NEW_IMAGE" \
        '.Spec.TaskTemplate.ContainerSpec.Image = $img')")

echo "Update initiated: $(echo "$UPDATE_RESPONSE" | jq '.')"
```

## Step 3: Monitor Rolling Update Progress

```bash
#!/bin/bash
# monitor-update.sh - Monitor rolling update progress

SERVICE_NAME="${1:?Specify service name}"
EXPECTED_REPLICAS="${2:-6}"

echo "Monitoring update for: $SERVICE_NAME"

while true; do
    # Get current status
    RUNNING=$(docker service ps "$SERVICE_NAME" \
        --filter "desired-state=running" \
        --format '{{.CurrentState}}' | \
        grep -c "Running")

    PREPARING=$(docker service ps "$SERVICE_NAME" \
        --filter "desired-state=running" \
        --format '{{.CurrentState}}' | \
        grep -c "Preparing")

    FAILED=$(docker service ps "$SERVICE_NAME" \
        --format '{{.CurrentState}}' | \
        grep -c "Failed")

    echo "[$(date '+%H:%M:%S')] Running: $RUNNING/$EXPECTED_REPLICAS | Preparing: $PREPARING | Failed: $FAILED"

    # Check if update is complete
    if [ "$RUNNING" -eq "$EXPECTED_REPLICAS" ] && [ "$PREPARING" -eq 0 ]; then
        echo "Update complete! All $EXPECTED_REPLICAS replicas running."
        break
    fi

    # Check for failure
    if [ "$FAILED" -gt 0 ]; then
        echo "ERROR: $FAILED tasks failed. Rolling back..."
        docker service rollback "$SERVICE_NAME"
        exit 1
    fi

    sleep 5
done

# Verify all tasks are healthy
echo "Verifying health checks..."
sleep 30

HEALTHY=$(docker service ps "$SERVICE_NAME" \
    --filter "desired-state=running" \
    --format '{{.CurrentState}}' | \
    grep -c "Running")

if [ "$HEALTHY" -eq "$EXPECTED_REPLICAS" ]; then
    echo "✓ All replicas healthy. Update successful!"
else
    echo "WARNING: Expected $EXPECTED_REPLICAS healthy replicas, found $HEALTHY"
fi
```

## Step 4: Manual Rollback

```bash
# Rollback via Docker CLI
docker service rollback myapp_api

# Rollback via Portainer API
curl -X POST \
    -H "X-API-Key: $API_KEY" \
    "$PORTAINER_URL/api/endpoints/1/docker/services/myapp_api/update?version=$VERSION&rollback=true"

# View rollback history
docker service ps myapp_api --no-trunc
```

## Step 5: Zero-Downtime Verification

```bash
#!/bin/bash
# verify-zero-downtime.sh - Test that rolling update has no downtime

TARGET_URL="https://api.yourdomain.com/health"
ERRORS=0
TOTAL=0

echo "Starting continuous health checks during update..."
echo "Press Ctrl+C to stop"

while true; do
    TOTAL=$((TOTAL + 1))
    HTTP_STATUS=$(curl -s -o /dev/null -w "%{http_code}" --max-time 2 "$TARGET_URL")

    if [ "$HTTP_STATUS" != "200" ]; then
        ERRORS=$((ERRORS + 1))
        echo "[$(date '+%H:%M:%S')] ERROR: HTTP $HTTP_STATUS (total errors: $ERRORS/$TOTAL)"
    else
        # Show status every 10 checks
        if [ $((TOTAL % 10)) -eq 0 ]; then
            echo "[$(date '+%H:%M:%S')] OK: $TOTAL checks, $ERRORS errors (${ERRORS}% error rate)"
        fi
    fi

    sleep 0.5
done
```

## Step 6: Portainer Update UI Walkthrough

In Portainer's Services view during a rolling update:

1. **Service details** shows update progress:
   - Current image tag vs new image tag
   - Number of tasks in each state

2. **Service tasks** tab shows individual replica states:
   - `Running` (old version being replaced)
   - `Preparing` (new version starting)
   - `Running` (new version healthy)

3. **Update failure** automatically triggers rollback if configured

## Conclusion

Docker Swarm rolling updates with Portainer provide true zero-downtime deployments. The `start-first` order ensures new replicas are healthy before old ones are stopped. Health checks prevent unhealthy replicas from being marked ready. Automatic rollback triggers if failures exceed the threshold. Portainer's Services view gives real-time visibility into the update progress, making it easy to monitor and intervene if needed.
