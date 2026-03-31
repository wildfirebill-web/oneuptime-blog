# How to Upgrade Portainer CE on Docker Swarm - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker Swarm, Upgrade, DevOps, Container

Description: Learn how to upgrade Portainer Community Edition deployed as a Docker Swarm service to the latest version with minimal downtime.

---

When Portainer is deployed as a Docker Swarm service, upgrading involves updating the service image rather than manually stopping and removing a container. Docker Swarm handles the rollout automatically.

## Prerequisites

- Portainer CE deployed as a Docker Swarm service
- Manager node access
- `portainer_data` volume on the manager node

## Back Up Before Upgrading

```bash
# Run on the Swarm manager node where Portainer data lives

docker run --rm \
  -v portainer_data:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/portainer_backup_$(date +%Y%m%d).tar.gz -C /data .
```

## Check Current Service Name

```bash
# List running Swarm services to find Portainer's service name
docker service ls --filter name=portainer
```

## Option 1: Update the Service Image

If Portainer is running as a Swarm service, update it directly:

```bash
# Update the Portainer service to the latest CE image
# --image sets the new image
# --force forces a redeployment even if the image tag hasn't changed
docker service update \
  --image portainer/portainer-ce:latest \
  --force \
  portainer
```

## Option 2: Redeploy the Stack

If Portainer was deployed via a Docker stack file, update the image tag and redeploy:

```yaml
# portainer-stack.yml
version: "3.8"

services:
  portainer:
    image: portainer/portainer-ce:latest  # Update this tag
    command: -H tcp://tasks.agent:9001 --tlsskipverify
    ports:
      - "9443:9443"
      - "9000:9000"
      - "8000:8000"
    volumes:
      - portainer_data:/data
    networks:
      - agent_network
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints:
          - node.role == manager

  agent:
    image: portainer/agent:latest
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /var/lib/docker/volumes:/var/lib/docker/volumes
    networks:
      - agent_network
    deploy:
      mode: global

networks:
  agent_network:
    driver: overlay
    attachable: true

volumes:
  portainer_data:
```

```bash
# Redeploy the updated stack
docker stack deploy -c portainer-stack.yml portainer
```

## Monitor the Update

```bash
# Watch the service update progress
docker service ps portainer --no-trunc

# Check for any failed tasks
docker service ps portainer --filter desired-state=running
```

## Verify the Upgrade

```bash
# Confirm the running task uses the new image
docker service inspect portainer --format '{{.Spec.TaskTemplate.ContainerSpec.Image}}'
```

Log in to `https://<manager-ip>:9443` and navigate to **Settings > About** to confirm the version number.

---

*Ensure your Swarm cluster and Portainer stay available with [OneUptime](https://oneuptime.com) uptime monitoring.*
