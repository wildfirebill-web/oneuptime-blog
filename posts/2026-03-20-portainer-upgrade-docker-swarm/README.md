# How to Upgrade Portainer CE on Docker Swarm

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Upgrade, Docker-swarm, Update

Description: A guide to upgrading Portainer CE deployed on Docker Swarm, covering service updates and maintaining Portainer agent connectivity.

## Overview

Upgrading Portainer CE on Docker Swarm involves updating the Portainer service image. The upgrade process leverages Docker Swarm's rolling update capability while ensuring the Portainer data volume is preserved. This guide covers the complete upgrade process for Swarm deployments.

## Portainer Swarm Architecture

In a Docker Swarm deployment, Portainer typically runs as:
- **Portainer Server**: A service constrained to manager nodes
- **Portainer Agent**: A global service on all nodes

## Step 1: Backup Portainer Data

```bash
# On the manager node where Portainer is running

# Find which node is running Portainer
docker service ps portainer --filter desired-state=running

# SSH to that node and backup
docker run --rm \
  -v portainer_data:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/portainer-swarm-backup-$(date +%Y%m%d).tar.gz -C /data .
```

## Step 2: Pull New Image on All Swarm Nodes

```bash
# Pull on all nodes (run on manager - Swarm will pull as needed)
# Or manually pull on each node:
for node in $(docker node ls -q); do
  docker node inspect $node --format '{{.Status.Addr}}' | xargs -I{} ssh {} "docker pull portainer/portainer-ce:latest"
done
```

## Step 3: Update the Portainer Service

```bash
# Update the Portainer service to new image
docker service update \
  --image portainer/portainer-ce:latest \
  --update-parallelism 1 \
  --update-delay 30s \
  portainer

# Monitor the update
docker service ps portainer
```

## Step 4: Update Portainer Agent

```bash
# Update the Portainer agent on all nodes
docker service update \
  --image portainer/agent:latest \
  portainer_agent

# Monitor agent updates across all nodes
docker service ps portainer_agent
```

## Step 5: Verify Upgrade

```bash
# Check service status
docker service ls | grep portainer

# Check all tasks are running
docker service ps portainer --filter desired-state=running

# Check logs
docker service logs portainer --tail 50
```

## Using Docker Stack for Managed Upgrades

If using `docker stack deploy`:

```yaml
# portainer-stack.yml
version: '3.8'

services:
  agent:
    image: portainer/agent:latest
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /var/lib/docker/volumes:/var/lib/docker/volumes
    networks:
      - agent-network
    deploy:
      mode: global
      placement:
        constraints: [node.platform.os == linux]

  portainer:
    image: portainer/portainer-ce:latest    # Update version here
    command: -H tcp://tasks.agent:9001 --tlsskipverify
    ports:
      - "9443:9443"
      - "9000:9000"
      - "8000:8000"
    volumes:
      - portainer_data:/data
    networks:
      - agent-network
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints: [node.role == manager]

networks:
  agent-network:
    driver: overlay
    attachable: true

volumes:
  portainer_data:
```

```bash
# Re-deploy the stack to update
docker stack deploy -c portainer-stack.yml portainer
```

## Conclusion

Upgrading Portainer on Docker Swarm is handled through Docker's service update mechanism, which provides controlled, rolling updates. Always backup the data volume before upgrading, update both the Portainer server service and the agent service, and verify the upgrade by checking service status and the Portainer UI. The Swarm service update approach is the recommended method for production Swarm environments.
