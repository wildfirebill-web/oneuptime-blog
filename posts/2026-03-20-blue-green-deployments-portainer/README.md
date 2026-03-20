# How to Implement Blue-Green Deployments with Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Blue-Green Deployment, CI/CD, Zero Downtime, DevOps

Description: Implement zero-downtime blue-green deployments using Docker networks and Traefik with Portainer for instant traffic switching.

## Introduction

Blue-green deployment maintains two identical environments: Blue (current production) and Green (new version). Traffic is routed to Blue while Green is deployed and tested. When ready, traffic switches instantly to Green - achieving zero-downtime deployments with an instant rollback capability. This guide implements blue-green with Docker and Portainer.

## Architecture

```text
           Traffic Router (Traefik)
                  │
         ┌────────┴────────┐
         ▼
    [ACTIVE: Blue]     [Standby: Green]
    app-blue:8080      app-green:8080
    v1.0.0             v1.1.0 (being tested)
```

## Step 1: Create the Blue-Green Stack

```yaml
# docker-compose.yml - Blue-Green deployment setup

version: "3.8"

networks:
  # Network for traffic router
  public:
    external: true
  # Internal network shared by both environments
  app_internal:
    driver: bridge

volumes:
  shared_data:

services:
  # Traffic router - Traefik
  traefik:
    image: traefik:v3.0
    container_name: traefik
    restart: unless-stopped
    ports:
      - "80:80"
      - "8080:8080"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    command:
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
    networks:
      - public
      - app_internal

  # Blue environment (current production)
  app_blue:
    image: myapp:1.0.0
    container_name: app_blue
    restart: unless-stopped
    environment:
      - VERSION=blue
      - COLOR=blue
    networks:
      - app_internal
    labels:
      - "traefik.enable=true"
      # Initially active - receives all traffic
      - "traefik.http.routers.app.rule=Host(`app.yourdomain.com`)"
      - "traefik.http.routers.app.entrypoints=web"
      - "traefik.http.services.app.loadbalancer.server.port=8080"

  # Green environment (new version, initially inactive)
  app_green:
    image: myapp:1.1.0
    container_name: app_green
    restart: unless-stopped
    environment:
      - VERSION=green
      - COLOR=green
    networks:
      - app_internal
    labels:
      # NOT enabled - won't receive production traffic
      - "traefik.enable=false"
      # Can be tested directly on a different domain
      - "traefik.http.routers.app-green-test.rule=Host(`green.yourdomain.com`)"
      - "traefik.http.routers.app-green-test.entrypoints=web"
      - "traefik.http.services.app-green-test.loadbalancer.server.port=8080"
```

## Step 2: Deploy via Portainer Stack

1. In Portainer, create a new stack named `app-production`
2. Paste the docker-compose above
3. Deploy the stack
4. Verify Blue is receiving traffic at `http://app.yourdomain.com`
5. Test Green at `http://green.yourdomain.com`

## Step 3: Traffic Switch Script

```bash
#!/bin/bash
# blue-green-switch.sh - Switch traffic between blue and green

STACK_NAME="app-production"
PORTAINER_URL="https://portainer.yourdomain.com"
PORTAINER_API_KEY="your-portainer-api-key"

# Determine current active environment
CURRENT=$(docker inspect app_blue | jq -r '.[0].Config.Labels."traefik.enable"')

if [ "$CURRENT" = "true" ]; then
    # Blue is active, switch to Green
    ACTIVATE="app_green"
    DEACTIVATE="app_blue"
    echo "Switching: Blue → Green"
else
    # Green is active, switch to Blue
    ACTIVATE="app_blue"
    DEACTIVATE="app_green"
    echo "Switching: Green → Blue"
fi

# Update labels via Docker
docker container stop "$DEACTIVATE"
docker container rm "$DEACTIVATE"

# Or use Portainer API to update the stack
# This is cleaner as it updates the stack definition
echo "Traffic switched to $ACTIVATE"
```

## Step 4: Automated Blue-Green with Portainer Webhooks

```bash
#!/bin/bash
# deploy-green.sh - Deploy new version to green and run tests

NEW_VERSION="$1"
PORTAINER_URL="https://portainer.yourdomain.com"
API_KEY="your-portainer-api-key"

echo "=== Starting Blue-Green Deployment ==="
echo "Deploying version: $NEW_VERSION to Green"

# Step 1: Pull new image
docker pull "myapp:$NEW_VERSION"

# Step 2: Update Green container with new image
docker stop app_green 2>/dev/null
docker rm app_green 2>/dev/null

docker run -d \
  --name app_green \
  --network app_internal \
  --label "traefik.enable=false" \
  --label "traefik.http.routers.app-green-test.rule=Host(\`green.yourdomain.com\`)" \
  --label "traefik.http.services.app-green-test.loadbalancer.server.port=8080" \
  -e VERSION=green \
  "myapp:$NEW_VERSION"

echo "Green deployed. Running health checks..."
sleep 10

# Step 3: Health check Green
HEALTH=$(curl -s -o /dev/null -w "%{http_code}" http://green.yourdomain.com/health)
if [ "$HEALTH" != "200" ]; then
    echo "ERROR: Green health check failed (HTTP $HEALTH)"
    exit 1
fi

echo "Green is healthy. Running integration tests..."

# Step 4: Run integration tests against Green
if ! ./run-integration-tests.sh http://green.yourdomain.com; then
    echo "ERROR: Integration tests failed"
    exit 1
fi

echo "All tests passed. Switching traffic to Green..."

# Step 5: Switch traffic
docker stop app_blue

docker run -d \
  --name app_blue_old \
  --network app_internal \
  --label "traefik.enable=false" \
  $(docker inspect app_blue --format='{{.Config.Image}}') || true

# Enable Green as active
docker label app_green traefik.enable=true
docker label app_green "traefik.http.routers.app.rule=Host(\`app.yourdomain.com\`)"

echo "=== Deployment Complete ==="
echo "Green (v$NEW_VERSION) is now serving production traffic"
echo "Blue (previous version) is stopped (rollback available)"
```

## Step 5: Instant Rollback

```bash
#!/bin/bash
# rollback.sh - Instantly revert to Blue

echo "=== ROLLBACK: Reverting to Blue ==="

# Disable Green
docker update --label-add traefik.enable=false app_green

# Enable Blue
docker start app_blue
docker update \
  --label-add traefik.enable=true \
  --label-add "traefik.http.routers.app.rule=Host(\`app.yourdomain.com\`)" \
  app_blue

echo "Rollback complete. Blue is now serving traffic."
```

## Step 6: Validate Deployment in Portainer

After switching:
1. Check Portainer **Containers** view - both containers should show as running
2. Verify Traefik dashboard shows the correct active route
3. Monitor container logs in Portainer for errors

```bash
# Validate traffic is flowing to correct container
curl -I http://app.yourdomain.com
# Check X-Server or other headers to identify which version is active

# Check container stats in Portainer
# Containers > app_green > Stats
# Should show increasing request count
```

## Conclusion

Blue-green deployments with Docker, Traefik, and Portainer give you zero-downtime releases with instant rollback capability. The pattern is simple: keep both versions running, test the new version before switching traffic, then switch instantly. Portainer makes it easy to manage both containers, view logs, and monitor the transition. The entire process can be automated in a CI/CD pipeline using Portainer's webhook API.
