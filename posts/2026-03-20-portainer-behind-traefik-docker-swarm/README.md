# How to Set Up Portainer Behind Traefik on Docker Swarm

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Traefik, Docker Swarm, Reverse Proxy, HTTPS

Description: Deploy Portainer and Traefik as Swarm services with automatic HTTPS and proper routing across a multi-node Docker Swarm cluster.

## Introduction

Docker Swarm requires a different approach to Traefik configuration compared to standalone Docker. Services are deployed via stacks, labels must be on the `deploy` block, and Portainer needs access to the Docker socket on the manager node. This guide covers the full deployment.

## Prerequisites

- Docker Swarm initialized (`docker swarm init`)
- A domain pointing to your manager node
- Ports 80 and 443 open on all manager nodes

## Step 1: Create Required Secrets and Configs

Store sensitive data in Docker Secrets:

```bash
# Create Let's Encrypt account credentials as a secret

echo "admin@example.com" | docker secret create acme_email -

# Create an empty acme.json for certificate storage
touch acme.json && chmod 600 acme.json
```

## Step 2: Deploy the Traefik Stack

Create `traefik-stack.yml`:

```yaml
version: "3.8"

services:
  traefik:
    image: traefik:v3.0
    command:
      - "--providers.docker=true"
      - "--providers.docker.swarmMode=true"           # Enable Swarm mode
      - "--providers.docker.exposedByDefault=false"
      - "--providers.docker.network=proxy"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.web.http.redirections.entryPoint.to=websecure"
      - "--entrypoints.web.http.redirections.entryPoint.scheme=https"
      - "--entrypoints.websecure.address=:443"
      - "--certificatesresolvers.letsencrypt.acme.email=admin@example.com"
      - "--certificatesresolvers.letsencrypt.acme.storage=/letsencrypt/acme.json"
      - "--certificatesresolvers.letsencrypt.acme.tlschallenge=true"
    ports:
      - target: 80
        published: 80
        mode: host    # host mode avoids NAT, needed for real client IPs
      - target: 443
        published: 443
        mode: host
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - traefik_certs:/letsencrypt
    networks:
      - proxy
    deploy:
      placement:
        constraints:
          - node.role == manager   # Must run on manager to read Swarm state
      restart_policy:
        condition: on-failure

  portainer:
    image: portainer/portainer-ce:latest
    command:
      - "--http-enabled"
      - "--trusted-origins=https://portainer.example.com"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data
    networks:
      - proxy
    deploy:
      placement:
        constraints:
          - node.role == manager   # Needs Docker socket on manager
      labels:
        - "traefik.enable=true"
        - "traefik.http.routers.portainer.rule=Host(`portainer.example.com`)"
        - "traefik.http.routers.portainer.entrypoints=websecure"
        - "traefik.http.routers.portainer.tls.certresolver=letsencrypt"
        # In Swarm mode, port must be specified on the service
        - "traefik.http.services.portainer.loadbalancer.server.port=9000"
      restart_policy:
        condition: on-failure

networks:
  proxy:
    driver: overlay
    attachable: true   # Allow non-Swarm containers to join

volumes:
  portainer_data:
  traefik_certs:
```

## Step 3: Deploy the Stack

```bash
# Deploy Traefik and Portainer together
docker stack deploy -c traefik-stack.yml traefik

# Watch service startup
docker service ls

# Check Traefik service logs
docker service logs traefik_traefik --follow
```

## Step 4: Verify Everything is Working

```bash
# List all running services
docker stack services traefik

# Inspect Portainer service
docker service inspect traefik_portainer --pretty

# Test HTTPS endpoint
curl -I https://portainer.example.com
```

## Important Swarm-Specific Notes

### Labels Must Be in deploy.labels

In Swarm mode, container-level labels are ignored by Traefik. Labels **must** be placed under `deploy.labels`:

```yaml
    deploy:
      labels:                          # CORRECT for Swarm
        - "traefik.enable=true"
      # NOT under top-level labels:   # WRONG - ignored in Swarm
```

### Overlay Network Is Required

Traefik and Portainer must share an overlay network. The `proxy` network must be created with the `overlay` driver, not `bridge`:

```bash
docker network create --driver overlay --attachable proxy
```

## Conclusion

Running Portainer behind Traefik on Docker Swarm provides a production-grade setup with automatic SSL, Swarm-aware routing, and centralized certificate management. The key differences from standalone mode are the `swarmMode` provider setting, `deploy.labels` placement, and the overlay network requirement.
