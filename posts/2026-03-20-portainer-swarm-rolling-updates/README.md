# How to Set Up Rolling Update Policies for Swarm Services in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker Swarm, Rolling Updates, Deployment, Zero Downtime

Description: Configure rolling update policies for Docker Swarm services in Portainer to achieve zero-downtime deployments.

## Introduction

Rolling updates deploy new versions of a service gradually, replacing old tasks with new ones while maintaining service availability. Portainer exposes Docker Swarm's update configuration through both its UI and stack files.

## Rolling Update Configuration

```yaml
# rolling-update-stack.yml

version: '3.8'

services:
  web:
    image: myapp:latest
    deploy:
      replicas: 6
      update_config:
        parallelism: 2       # Update 2 replicas at a time
        delay: 30s           # Wait 30s between each batch
        failure_action: rollback  # Auto-rollback on failure
        monitor: 60s         # Monitor for 60s after each update
        max_failure_ratio: 0.3  # Allow up to 30% failure rate
        order: start-first   # Start new before stopping old
      rollback_config:
        parallelism: 1       # Rollback 1 at a time
        delay: 10s
        failure_action: pause
        monitor: 60s
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 15s
      timeout: 5s
      retries: 3
      start_period: 30s
```

## Triggering Updates via Portainer

### Via Portainer UI

1. Go to **Swarm > Services > Your Service**
2. Click **Edit**
3. Update the image tag to the new version
4. Click **Update the service**

Portainer shows the update progress in real-time.

### Via Portainer API

```bash
# Update a service image via API
SERVICE_ID=$(docker service ls --filter name=myapp_web -q)

curl -X POST \
  -H "X-API-Key: your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "Name": "myapp_web",
    "TaskTemplate": {
      "ContainerSpec": {
        "Image": "myapp:1.2.0"
      }
    },
    "UpdateConfig": {
      "Parallelism": 2,
      "Delay": 30000000000,
      "FailureAction": "rollback",
      "Monitor": 60000000000,
      "Order": "start-first"
    }
  }' \
  "https://portainer.example.com/api/endpoints/1/docker/services/$SERVICE_ID/update?version=$VERSION"
```

## Monitoring Update Progress

```bash
# Watch the update progress
watch -n 2 docker service ps myapp_web

# Or via Portainer API
curl -s -H "X-API-Key: your-api-key" \
  "https://portainer.example.com/api/endpoints/1/docker/services/$SERVICE_ID" \
  | python3 -c "
import sys, json
s = json.load(sys.stdin)
update = s['UpdateStatus']
print(f'State: {update[\"State\"]}')
print(f'Message: {update[\"Message\"]}')
"
```

## Blue-Green Deployment Pattern

```yaml
# blue-green-stack.yml
version: '3.8'

services:
  # Blue: currently receiving traffic
  blue:
    image: myapp:1.0.0
    deploy:
      replicas: 3
    networks:
      - frontend
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.app.rule=Host(`myapp.example.com`)"

  # Green: new version, not yet receiving traffic
  green:
    image: myapp:2.0.0
    deploy:
      replicas: 3
    networks:
      - frontend
    labels:
      - "traefik.enable=false"  # Not routing traffic yet

networks:
  frontend:
    driver: overlay
```

```bash
# Switch traffic from blue to green:
# 1. Enable green
docker service update --label-add traefik.enable=true myapp_green

# 2. Disable blue
docker service update --label-add traefik.enable=false myapp_blue

# 3. After verification, remove blue
docker service rm myapp_blue
```

## Conclusion

Rolling updates in Docker Swarm via Portainer enable zero-downtime deployments with automatic failure detection and rollback. The `update_config` parameters give fine-grained control over update speed, failure tolerance, and rollback behavior. Using `start-first` order ensures new tasks are running and healthy before old tasks are removed, maintaining continuous service availability throughout the update.
