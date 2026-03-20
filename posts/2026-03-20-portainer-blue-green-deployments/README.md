# How to Implement Blue-Green Deployments with Portainer - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Blue-Green Deployment, Zero Downtime, Docker, Traefik, CI/CD

Description: Learn how to implement blue-green deployments with Portainer to achieve zero-downtime releases by running two identical environments and switching traffic between them.

---

Blue-green deployment maintains two identical production environments - "blue" (current) and "green" (new version). Traffic switches instantly between them, enabling zero-downtime releases and instant rollbacks.

## Blue-Green with Traefik

Use Traefik labels to route traffic to the active environment:

```yaml
version: "3.8"

services:
  traefik:
    image: traefik:v3.0
    command:
      - --api.insecure=true
      - --providers.docker=true
      - --providers.docker.exposedbydefault=false
      - --entrypoints.web.address=:80
    ports:
      - "80:80"
      - "8080:8080"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks:
      - proxy_net

  # Blue environment (current version)
  api-blue:
    image: myregistry.example.com/my-app:v1.4.0
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.api.rule=Host(`api.example.com`)"
      - "traefik.http.routers.api.service=api-blue"   # Point to blue
      - "traefik.http.services.api-blue.loadbalancer.server.port=3000"
    networks:
      - proxy_net

  # Green environment (new version - deployed but not receiving traffic)
  api-green:
    image: myregistry.example.com/my-app:v1.5.0
    labels:
      - "traefik.enable=true"
      # Green is NOT the active router target yet
      - "traefik.http.services.api-green.loadbalancer.server.port=3000"
    networks:
      - proxy_net

networks:
  proxy_net:
    driver: bridge
```

## Switching Traffic

To switch from blue to green, update the router's service label:

```bash
# Update the stack: change the router service from api-blue to api-green

# In Portainer, edit the stack and update this label:
# "traefik.http.routers.api.service=api-green"

# Or use docker service update (Swarm mode):
docker service update \
  --label-add "traefik.http.routers.api.service=api-green" \
  my-stack_api-blue
```

Traefik detects the label change and reroutes traffic within seconds - no downtime.

## Rolling Back

If the green environment has issues, switch back to blue instantly:

```bash
# Revert the router to blue
# Edit stack in Portainer and update:
# "traefik.http.routers.api.service=api-blue"
```

Blue is still running with the previous version, so the rollback is immediate.

## Automated Blue-Green Switch Script

```bash
#!/bin/bash
# blue-green-switch.sh

TRAEFIK_API="http://traefik:8080"
ACTIVE=$(curl -s "$TRAEFIK_API/api/http/routers/api@docker" | jq -r '.service')

if [[ "$ACTIVE" == *"blue"* ]]; then
  NEW="green"
  OLD="blue"
else
  NEW="blue"
  OLD="green"
fi

echo "Switching from $OLD to $NEW..."

# Update the stack file and redeploy via Portainer API
TOKEN=$(curl -s -X POST https://portainer.example.com/api/auth \
  -d '{"Username":"admin","Password":"pass"}' \
  -H 'Content-Type: application/json' | jq -r .jwt)

# Update stack (assumes you store the compose content with the new target)
curl -s -X PUT -H "Authorization: Bearer $TOKEN" \
  https://portainer.example.com/api/stacks/1 \
  -H "Content-Type: application/json" \
  -d "{\"stackFileContent\": \"$(cat docker-compose-$NEW.yml | python3 -c 'import sys,json; print(json.dumps(sys.stdin.read()))')\", \"env\": []}"

echo "Traffic now routing to $NEW"
```

## Smoke Testing the Green Environment

Before switching, test the green deployment directly:

```bash
# Access green environment on its direct port (not the main domain)
curl -H "Host: api-green.internal" http://localhost:3001/health
```

Only switch traffic after confirming green is healthy.
