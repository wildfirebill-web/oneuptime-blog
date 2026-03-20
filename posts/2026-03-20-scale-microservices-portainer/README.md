# How to Scale Individual Microservices in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Microservices, Scaling, Docker Swarm, Auto-Scaling

Description: Scale individual microservices horizontally in Portainer using Docker Swarm replica management, resource limits, and auto-scaling triggers.

## Introduction

One of the key benefits of microservices is the ability to scale individual components independently. If your order processing is under load but your user service is idle, you scale only the order service. Portainer makes this straightforward with its visual scaling interface. This guide covers manual scaling, resource constraints, and auto-scaling strategies.

## Step 1: Scale Services in Portainer (Visual)

For Docker Swarm services:
1. Navigate to **Services** in Portainer
2. Click on a service (e.g., `myapp_order_service`)
3. Find the **Replicas** field
4. Change the number and click **Apply changes**

## Step 2: Deploy a Scalable Swarm Stack

```yaml
# docker-compose.yml - Scalable microservice stack
version: "3.8"

networks:
  app_overlay:
    driver: overlay
    attachable: true

services:
  # API Gateway (Traefik)
  traefik:
    image: traefik:v3.0
    deploy:
      # Single instance per manager node
      mode: global
      placement:
        constraints:
          - node.role == manager
    ports:
      - "80:80"
      - "8080:8080"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    command:
      - "--providers.docker.swarmMode=true"
      - "--providers.docker.network=app_overlay"
      - "--providers.docker.exposedByDefault=false"
      - "--entrypoints.web.address=:80"
    networks:
      - app_overlay

  # User Service - scales independently
  user_service:
    image: myapp/user-service:latest
    deploy:
      replicas: 2
      # Resource constraints per replica
      resources:
        limits:
          cpus: "0.5"       # Max 50% of one CPU
          memory: 256M
        reservations:
          cpus: "0.1"       # Guaranteed minimum
          memory: 128M
      # Update strategy
      update_config:
        parallelism: 1
        delay: 10s
        order: start-first    # Start new before stopping old
        failure_action: rollback
      # Auto-restart policy
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
        window: 120s
      labels:
        - "traefik.enable=true"
        - "traefik.http.routers.users.rule=PathPrefix(`/api/users`)"
        - "traefik.http.services.users.loadbalancer.server.port=8002"
    environment:
      - DATABASE_URL=postgresql://user:pass@user_db:5432/userdb
    networks:
      - app_overlay

  # Order Service - scales independently (more compute needed)
  order_service:
    image: myapp/order-service:latest
    deploy:
      replicas: 4
      resources:
        limits:
          cpus: "1.0"
          memory: 512M
        reservations:
          cpus: "0.25"
          memory: 256M
      placement:
        # Spread across multiple nodes
        preferences:
          - spread: node.labels.region
      labels:
        - "traefik.enable=true"
        - "traefik.http.routers.orders.rule=PathPrefix(`/api/orders`)"
        - "traefik.http.services.orders.loadbalancer.server.port=8003"
    networks:
      - app_overlay
```

## Step 3: Scale via Docker CLI

```bash
# Scale individual service
docker service scale myapp_order_service=8

# Scale multiple services at once
docker service scale \
  myapp_user_service=3 \
  myapp_order_service=6 \
  myapp_product_service=4

# Verify scaling
docker service ls
docker service ps myapp_order_service
```

## Step 4: Portainer API for Programmatic Scaling

```bash
#!/bin/bash
# scale-service.sh - Scale via Portainer API

PORTAINER_URL="https://portainer.yourdomain.com"
API_KEY="your-portainer-api-key"
STACK_ID="1"
SERVICE_ID="myapp_order_service"
NEW_REPLICAS=6

# Get current service details
SERVICE=$(curl -s \
  -H "X-API-Key: $API_KEY" \
  "$PORTAINER_URL/api/endpoints/1/docker/services/$SERVICE_ID")

# Update replica count
curl -X POST \
  -H "X-API-Key: $API_KEY" \
  -H "Content-Type: application/json" \
  "$PORTAINER_URL/api/endpoints/1/docker/services/$SERVICE_ID/update?version=$(echo $SERVICE | jq '.Version.Index')" \
  -d "$(echo $SERVICE | jq --arg r "$NEW_REPLICAS" '.Spec.Mode.Replicated.Replicas = ($r|tonumber)')"

echo "Scaled $SERVICE_ID to $NEW_REPLICAS replicas"
```

## Step 5: Auto-Scaling with Custom Metrics

```bash
#!/bin/bash
# autoscale.sh - Scale based on metrics

PROMETHEUS_URL="http://prometheus:9090"
SERVICE="myapp_order_service"

# Query CPU usage per replica
CPU_USAGE=$(curl -s "$PROMETHEUS_URL/api/v1/query" \
  --data-urlencode "query=avg(rate(container_cpu_usage_seconds_total{container_label_com_docker_swarm_service_name=\"$SERVICE\"}[5m])) * 100" \
  | jq -r '.data.result[0].value[1]')

# Current replicas
CURRENT_REPLICAS=$(docker service ls --filter name=$SERVICE --format '{{.Replicas}}' | cut -d'/' -f1)

echo "CPU Usage: ${CPU_USAGE}% | Replicas: ${CURRENT_REPLICAS}"

# Scale up if CPU > 70%
if (( $(echo "$CPU_USAGE > 70" | bc -l) )); then
    NEW_REPLICAS=$((CURRENT_REPLICAS + 2))
    MAX_REPLICAS=20

    if [ "$NEW_REPLICAS" -le "$MAX_REPLICAS" ]; then
        echo "Scaling UP: $CURRENT_REPLICAS → $NEW_REPLICAS replicas"
        docker service scale "$SERVICE=$NEW_REPLICAS"
    fi
fi

# Scale down if CPU < 20%
if (( $(echo "$CPU_USAGE < 20" | bc -l) )); then
    NEW_REPLICAS=$((CURRENT_REPLICAS - 1))
    MIN_REPLICAS=2

    if [ "$NEW_REPLICAS" -ge "$MIN_REPLICAS" ]; then
        echo "Scaling DOWN: $CURRENT_REPLICAS → $NEW_REPLICAS replicas"
        docker service scale "$SERVICE=$NEW_REPLICAS"
    fi
fi
```

```bash
# Run auto-scaler every 2 minutes
echo "*/2 * * * * /usr/local/bin/autoscale.sh" | crontab -
```

## Step 6: Deploy Docker Autoscaler (3rd Party)

```yaml
# docker-compose.yml - Docker Autoscaler (Orbica)
services:
  autoscaler:
    image: orbica/docker-autoscaler:latest
    container_name: docker_autoscaler
    restart: unless-stopped
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - DOCKER_HOST=unix:///var/run/docker.sock
      - AUTOSCALER_PROMETHEUS_URL=http://prometheus:9090
      - AUTOSCALER_MIN_REPLICAS=2
      - AUTOSCALER_MAX_REPLICAS=20
      - AUTOSCALER_TARGET_CPU=60  # Target 60% CPU
      - AUTOSCALER_SCALE_UP_COOLDOWN=60s
      - AUTOSCALER_SCALE_DOWN_COOLDOWN=300s
```

## Monitoring Scale Events in Portainer

1. Navigate to **Services** to see current replica counts
2. Click a service to see individual task health
3. Check **Service logs** for scale event messages

```bash
# View scale events
docker events --filter type=service --filter event=update

# View current service state
docker service inspect myapp_order_service --format='Replicas: {{.Spec.Mode.Replicated.Replicas}}'

# Check which nodes replicas are running on
docker service ps myapp_order_service
```

## Conclusion

Portainer makes horizontal scaling intuitive — just change the replica count in the UI. For production environments, combine manual scaling (for planned events like marketing campaigns) with auto-scaling scripts based on Prometheus metrics (for unexpected load spikes). Docker Swarm ensures replicas are distributed across nodes, and Traefik's load balancer automatically routes traffic to all healthy replicas. Resource limits prevent any single service from consuming all available CPU and memory.
