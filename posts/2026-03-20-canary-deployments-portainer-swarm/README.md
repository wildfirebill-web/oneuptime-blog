# How to Implement Canary Deployments with Portainer on Swarm

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker Swarm, Canary Deployment, CI/CD, Zero Downtime, DevOps

Description: Implement canary deployments on Docker Swarm with Portainer to gradually roll out new versions to a percentage of users.

## Introduction

Canary deployment gradually shifts traffic from the current version to a new version — starting with a small percentage (5-10%) and increasing as confidence grows. If issues arise, you roll back only the canary, limiting blast radius. Docker Swarm's replica system makes this straightforward. This guide shows you how using Portainer.

## How Canary Works with Docker Swarm

With 10 service replicas:
- Stable version: 9 replicas
- Canary version: 1 replica
- Traefik round-robins traffic: ~10% goes to canary

## Step 1: Deploy Base Service on Swarm

```bash
# Initialize Swarm if not done
docker swarm init

# Create overlay network
docker network create --driver overlay --attachable app_overlay
```

## Step 2: Deploy Stable Version as a Stack

In Portainer, create a new Swarm Stack:

```yaml
# stable-stack.yml - Stable production service
version: "3.8"

networks:
  app_overlay:
    external: true
  traefik_overlay:
    external: true

services:
  # Traefik load balancer
  traefik:
    image: traefik:v3.0
    deploy:
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
      - "--providers.docker.network=traefik_overlay"
      - "--providers.docker.exposedByDefault=false"
      - "--entrypoints.web.address=:80"
      - "--api.insecure=true"
    networks:
      - traefik_overlay

  # Stable application (current version)
  app_stable:
    image: myapp:1.0.0
    deploy:
      replicas: 9
      labels:
        - "traefik.enable=true"
        - "traefik.http.routers.app.rule=Host(`app.yourdomain.com`)"
        - "traefik.http.routers.app.entrypoints=web"
        - "traefik.http.services.app.loadbalancer.server.port=8080"
        # Traefik uses the same service name for all replicas
      update_config:
        parallelism: 2
        delay: 10s
        order: start-first
      rollback_config:
        parallelism: 1
      restart_policy:
        condition: on-failure
    networks:
      - app_overlay
      - traefik_overlay
    environment:
      - VERSION=stable-1.0.0
```

## Step 3: Deploy Canary Version

Add the canary service to the stack in Portainer:

```yaml
# canary-stack.yml - Deploy canary alongside stable
version: "3.8"

networks:
  app_overlay:
    external: true
  traefik_overlay:
    external: true

services:
  # Canary application (new version - 1 of 10 replicas)
  app_canary:
    image: myapp:1.1.0
    deploy:
      replicas: 1
      labels:
        - "traefik.enable=true"
        # Uses SAME router as stable - traffic distributed across all replicas
        - "traefik.http.routers.app.rule=Host(`app.yourdomain.com`)"
        - "traefik.http.routers.app.entrypoints=web"
        - "traefik.http.services.app.loadbalancer.server.port=8080"
      restart_policy:
        condition: on-failure
    networks:
      - app_overlay
      - traefik_overlay
    environment:
      - VERSION=canary-1.1.0
```

## Step 4: Gradually Increase Canary Traffic

```bash
#!/bin/bash
# canary-promote.sh - Gradually increase canary traffic

STABLE_SERVICE="mystack_app_stable"
CANARY_SERVICE="mystack_app_canary"

echo "Current state:"
docker service ls | grep -E "app_stable|app_canary"

# Phase 1: 10% canary (9 stable, 1 canary)
echo "Phase 1: 10% canary traffic"
docker service scale ${STABLE_SERVICE}=9 ${CANARY_SERVICE}=1

# Monitor for 10 minutes
sleep 600

# Check error rates before proceeding
ERROR_RATE=$(check_error_rate)
if [ "$ERROR_RATE" -gt 1 ]; then
    echo "Error rate too high ($ERROR_RATE%). Rolling back."
    docker service scale ${CANARY_SERVICE}=0
    exit 1
fi

# Phase 2: 25% canary
echo "Phase 2: 25% canary traffic"
docker service scale ${STABLE_SERVICE}=6 ${CANARY_SERVICE}=2

sleep 600

# Phase 3: 50% canary
echo "Phase 3: 50% canary traffic"
docker service scale ${STABLE_SERVICE}=5 ${CANARY_SERVICE}=5

sleep 600

# Phase 4: Full promotion
echo "Phase 4: Promoting canary to 100%"
docker service update \
  --image myapp:1.1.0 \
  --update-parallelism 2 \
  --update-delay 10s \
  ${STABLE_SERVICE}

# Remove canary
docker service scale ${CANARY_SERVICE}=0

echo "Canary deployment complete!"
```

## Step 5: Monitor Canary via Portainer + Prometheus

```yaml
# Add Prometheus metrics scraping for canary monitoring
services:
  prometheus:
    image: prom/prometheus:latest
    deploy:
      replicas: 1
    volumes:
      - /opt/prometheus:/etc/prometheus
    ports:
      - "9090:9090"
    networks:
      - app_overlay
```

```yaml
# prometheus.yml - Scrape Docker service metrics
scrape_configs:
  - job_name: 'app_stable'
    dns_sd_configs:
      - names: ['tasks.app_stable']
        type: A
        port: 8080
    metrics_path: /metrics
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance

  - job_name: 'app_canary'
    dns_sd_configs:
      - names: ['tasks.app_canary']
        type: A
        port: 8080
    metrics_path: /metrics
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance
```

## Step 6: Automatic Canary Analysis with Prometheus Queries

```bash
#!/bin/bash
# check-canary-health.sh - Automated canary analysis

PROM_URL="http://prometheus:9090"

# Check error rate for canary (5xx responses)
CANARY_ERROR_RATE=$(curl -s "$PROM_URL/api/v1/query" \
  --data-urlencode 'query=rate(http_requests_total{status=~"5..",job="app_canary"}[5m]) / rate(http_requests_total{job="app_canary"}[5m]) * 100' \
  | jq -r '.data.result[0].value[1]')

# Check P99 latency for canary
CANARY_P99=$(curl -s "$PROM_URL/api/v1/query" \
  --data-urlencode 'query=histogram_quantile(0.99, rate(http_request_duration_seconds_bucket{job="app_canary"}[5m]))' \
  | jq -r '.data.result[0].value[1]')

echo "Canary Error Rate: ${CANARY_ERROR_RATE}%"
echo "Canary P99 Latency: ${CANARY_P99}s"

# Decision logic
if (( $(echo "$CANARY_ERROR_RATE > 2" | bc -l) )); then
    echo "FAIL: Error rate too high. Rolling back canary."
    docker service scale mystack_app_canary=0
    exit 1
fi

if (( $(echo "$CANARY_P99 > 2" | bc -l) )); then
    echo "FAIL: Latency too high. Rolling back canary."
    docker service scale mystack_app_canary=0
    exit 1
fi

echo "PASS: Canary metrics are healthy. Proceeding."
```

## Viewing Canary in Portainer

1. Navigate to **Services** in Portainer (Swarm mode)
2. You'll see both `app_stable` and `app_canary` with replica counts
3. Click on a service to see individual task health
4. Use the **Scale** button to adjust replica counts
5. View task logs to compare behavior between versions

## Conclusion

Canary deployments on Docker Swarm with Portainer give you granular control over traffic percentage by adjusting replica ratios. The automated analysis script checks error rates and latency before promoting — stopping bad deployments before they affect all users. Portainer's Services view makes it easy to see the current state, scale services, and roll back if needed. This pattern is especially valuable for APIs where even small changes can have unexpected performance impacts.
