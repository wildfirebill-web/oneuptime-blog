# How to Hide Containers Using Labels in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: portainer, containers, labels, filtering, visibility, management

Description: A guide to hiding containers from the Portainer UI using Docker labels, useful for system containers, infrastructure services, or decluttering the interface.

## Overview

Portainer supports hiding containers from its UI by applying specific Docker labels. This is useful for hiding infrastructure containers (Portainer Agent, Traefik, monitoring agents) from the Portainer view, reducing noise in the UI, or restricting what non-admin users can see. This guide covers all methods for hiding containers using labels.

## Prerequisites

- Portainer CE or Business Edition
- Docker CLI access

## Understanding Portainer's Hide Label

Portainer respects the label `portainer.agent.secret` for authentication, but for hiding containers, it uses:

```
com.docker.compose.project=portainer  # Hides Portainer's own stack
```

More specifically, Portainer has a configurable "hide label" feature.

## Method 1: Hide Containers with portainer.hide Label

```bash
# Label to hide a container from Portainer UI
docker run -d \
  --name my-monitoring-agent \
  --label "hide=true" \
  --restart=always \
  grafana/agent:latest
```

## Method 2: Configure Portainer to Respect Hide Labels

In Portainer settings, you can configure which label value marks containers as hidden:

1. Navigate to **Settings** → **App Settings**  
2. Find **Hidden containers**
3. Enter the label filter (e.g., `hide=true`)
4. Click **Save**

Any container with this label will be hidden from the Portainer UI.

## Method 3: Hide Portainer Infrastructure Containers

```yaml
# docker-compose.yml — hide infrastructure services from Portainer
version: "3.8"
services:
  traefik:
    image: traefik:v3.0
    labels:
      - "hide=true"          # Hidden from Portainer
      - "traefik.enable=false"
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    restart: unless-stopped

  portainer-agent:
    image: portainer/agent:latest
    labels:
      - "hide=true"          # Hide agent from Portainer
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /var/lib/docker/volumes:/var/lib/docker/volumes
    restart: unless-stopped
```

## Method 4: Apply Labels to Running Containers

Docker labels cannot be changed on running containers without recreation. Use `docker-compose` or recreate:

```bash
# Add label when recreating a container
docker stop my-container
docker rm my-container

docker run -d \
  --name my-container \
  --label "hide=true" \
  my-image:latest
```

## Method 5: Using Multiple Hide Conditions

```bash
# Multiple criteria for hiding
docker run -d \
  --name infrastructure-service \
  --label "hide=true" \
  --label "infrastructure=true" \
  --label "managed-by=ops-team" \
  my-infra-image:latest
```

Configure Portainer to filter on `infrastructure=true` if you prefer.

## Listing Hidden Containers

Hidden containers still exist and run; they're just not shown in the UI:

```bash
# See all containers including hidden ones via Docker CLI
docker ps -a

# Filter containers with the hide label
docker ps -a --filter "label=hide=true"

# Remove the hide label by recreating without it
docker run -d --name my-container my-image:latest  # No --label "hide=true"
```

## Role-Based Visibility (Portainer BE)

Portainer Business Edition offers more granular visibility controls:

```bash
# Restrict endpoint access to specific teams
# Settings → Teams → Assign endpoint access

# Users only see containers in their allowed environments
# No label needed — access control at environment level
```

## Use Cases

| Use Case | Approach |
|---|---|
| Hide Portainer's own agent | Label `hide=true` on agent container |
| Hide monitoring stack | Label all monitoring containers |
| Hide CI/CD runners | Label runner containers |
| Multi-team environments | Use Portainer BE RBAC |
| Compliance auditing | Use access controls, not hiding |

## Conclusion

Hiding containers with labels is a practical way to reduce noise in the Portainer UI and prevent users from accidentally modifying infrastructure services. The `hide=true` label convention (configured in Portainer settings) provides a simple, declarative approach. For multi-team environments with security requirements, combine label-based hiding with Portainer BE's role-based access controls.
