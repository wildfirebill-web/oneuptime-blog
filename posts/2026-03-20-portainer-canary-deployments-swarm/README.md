# How to Implement Canary Deployments with Portainer on Swarm

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Canary Deployment, Docker Swarm, Zero Downtime, Traefik, CI/CD

Description: Learn how to implement canary deployments with Portainer on Docker Swarm to gradually roll out new versions to a small percentage of traffic before full release.

---

A canary deployment routes a small percentage of traffic to a new version while the rest continues to the stable version. This allows you to monitor the new version with real user traffic before committing to a full rollout.

## Canary with Traefik Weighted Load Balancing

Traefik supports weighted services, making canary deployments straightforward:

```yaml
version: "3.8"

services:
  traefik:
    image: traefik:v3.0
    command:
      - --api.insecure=true
      - --providers.docker=true
      - --providers.docker.swarmMode=true
      - --providers.docker.exposedbydefault=false
      - --entrypoints.web.address=:80
    ports:
      - "80:80"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    deploy:
      placement:
        constraints: [node.role == manager]
    networks:
      - proxy_net

  # Stable version (90% traffic)
  api-stable:
    image: myregistry.example.com/my-app:v1.4.0
    deploy:
      replicas: 9
      labels:
        - "traefik.enable=true"
        - "traefik.http.routers.api.rule=Host(`api.example.com`)"
        - "traefik.http.routers.api.service=weighted-api"
        - "traefik.http.services.api-stable.loadbalancer.server.port=3000"
        - "traefik.http.services.weighted-api.weighted.services[0].name=api-stable"
        - "traefik.http.services.weighted-api.weighted.services[0].weight=90"
        - "traefik.http.services.weighted-api.weighted.services[1].name=api-canary"
        - "traefik.http.services.weighted-api.weighted.services[1].weight=10"
    networks:
      - proxy_net

  # Canary version (10% traffic)
  api-canary:
    image: myregistry.example.com/my-app:v1.5.0
    deploy:
      replicas: 1
      labels:
        - "traefik.enable=true"
        - "traefik.http.services.api-canary.loadbalancer.server.port=3000"
    networks:
      - proxy_net

networks:
  proxy_net:
    driver: overlay
    attachable: true
```

The `weighted-api` service routes 90% of traffic to `api-stable` and 10% to `api-canary`.

## Gradual Traffic Shifting

Increase canary traffic over time as confidence grows:

```bash
# Phase 1: 10% canary
# Set: stable weight=90, canary weight=10

# Phase 2: After 30 minutes with no errors, increase to 30%
docker service update \
  --label-add "traefik.http.services.weighted-api.weighted.services[0].weight=70" \
  --label-add "traefik.http.services.weighted-api.weighted.services[1].weight=30" \
  my-stack_api-stable

# Phase 3: 50/50 split
# Phase 4: 100% canary (remove stable)
```

## Monitoring the Canary

Watch error rates and response times during the canary phase:

```bash
#!/bin/bash
# monitor-canary.sh

LOKI_URL="http://loki:3100"
THRESHOLD_ERROR_RATE=0.05  # 5% error rate triggers rollback

while true; do
  # Query error rate from Loki
  ERROR_RATE=$(curl -s "$LOKI_URL/loki/api/v1/query" \
    --data-urlencode 'query=sum(rate({service="api-canary"} |= "error" [5m])) / sum(rate({service="api-canary"} [5m]))' \
    | jq -r '.data.result[0].value[1] // "0"')

  echo "Canary error rate: $ERROR_RATE"

  if (( $(echo "$ERROR_RATE > $THRESHOLD_ERROR_RATE" | bc -l) )); then
    echo "ERROR RATE TOO HIGH — rolling back canary"
    # Remove canary weights and route 100% to stable
    docker service update \
      --label-add "traefik.http.services.weighted-api.weighted.services[0].weight=100" \
      --label-add "traefik.http.services.weighted-api.weighted.services[1].weight=0" \
      my-stack_api-stable
    break
  fi

  sleep 60
done
```

## Promoting the Canary to Production

Once confident, redirect all traffic to the canary and decommission the stable version:

```bash
# Phase: Full canary promotion
# Update stack to:
# 1. Set canary weight to 100, stable to 0
# 2. Rename canary to stable
# 3. Remove old stable service

# Or simply scale down the stable service:
docker service scale my-stack_api-stable=0

# Watch for issues, then remove it:
docker service rm my-stack_api-stable
```

## Canary for Database Migrations

For deployments that include database migrations, run the canary against a shadow database first:

```yaml
  api-canary:
    environment:
      DATABASE_URL: "postgresql://user:pass@postgres:5432/appdb_canary"  # Shadow DB
      # Run migrations against shadow DB first
```

After validating on shadow DB, migrate the production DB and promote.
