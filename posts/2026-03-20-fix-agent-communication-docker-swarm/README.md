# How to Fix Agent Communication Issues on Docker Swarm

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Troubleshooting, Docker Swarm, Agent, Networking, Overlay Network

Description: Learn how to diagnose and fix Portainer agent communication failures on Docker Swarm, including overlay network issues, service placement, and port configuration.

---

Running the Portainer Agent as a Swarm global service introduces overlay network communication that adds complexity beyond standalone Docker deployments. This guide covers the most common failure patterns.

## Correct Agent Deployment for Swarm

The agent must be deployed as a global service (one instance per node) and must have access to the Docker socket:

```bash
# Deploy the agent stack on your Swarm manager
docker stack deploy --compose-file agent-stack.yml portainer_agent
```

The agent stack YAML must include proper constraints and socket mounts:

```yaml
version: "3.8"

services:
  agent:
    image: portainer/agent:latest
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /var/lib/docker/volumes:/var/lib/docker/volumes
    networks:
      - portainer_agent_network
    environment:
      AGENT_CLUSTER_ADDR: tasks.agent   # DNS name of the agent service
    deploy:
      mode: global                       # Run on every node
      placement:
        constraints:
          - node.platform.os == linux

networks:
  portainer_agent_network:
    driver: overlay
    attachable: true
```

## Check Overlay Network Connectivity

```bash
# List nodes and verify all are active
docker node ls

# Check if the agent service has a task on every node
docker service ps portainer_agent_agent

# Look for tasks in "Running" state on each node
# "Failed" tasks indicate the agent cannot start on that node
```

## Diagnose Overlay Network Issues

```bash
# Check overlay network encryption status
docker network inspect portainer_agent_portainer_agent_network | grep -i encrypt

# Test connectivity between nodes
docker exec -it $(docker ps -q -f name=portainer_agent) \
  ping -c 3 tasks.portainer_agent_agent
```

## Firewall Ports for Swarm Overlay

All Swarm nodes need these ports open between each other:

```bash
# Required Swarm ports
sudo ufw allow 2377/tcp   # Swarm cluster management
sudo ufw allow 7946/tcp   # Node communication
sudo ufw allow 7946/udp   # Node communication
sudo ufw allow 4789/udp   # Overlay network traffic
sudo ufw allow 9001/tcp   # Portainer agent
```

## Verify AGENT_CLUSTER_ADDR

The `AGENT_CLUSTER_ADDR` must resolve to all agent task IPs. Using `tasks.<service-name>` in the overlay network achieves this automatically via Swarm DNS.

```bash
# Verify DNS resolution from within the agent container
docker exec -it $(docker ps -q -f name=portainer_agent) \
  nslookup tasks.portainer_agent_agent
# Should return multiple A records, one per node
```
