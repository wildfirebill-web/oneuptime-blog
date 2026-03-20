# How to Fix Agent Communication Issues on Docker Swarm

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Troubleshooting, Docker Swarm, Agent, Networking

Description: Troubleshoot and resolve Portainer Agent communication failures in Docker Swarm clusters, including overlay network issues, service placement constraints, and inter-node connectivity.

## Introduction

Running the Portainer Agent as a Docker Swarm service introduces additional complexity over standalone Docker deployments. Communication issues on Swarm often stem from overlay network configuration, node reachability, service placement, or port conflicts within the Swarm mesh routing.

## Step 1: Verify Swarm Cluster Health

```bash
# Check Swarm nodes
docker node ls

# Look for nodes in state "Down" or "Unreachable"
# All nodes should show "Ready" and "Active"

# Check node details
docker node inspect <node-id> --pretty
```

## Step 2: Deploy Portainer Agent as a Global Service

The Portainer Agent should run on **every** Swarm node as a global service:

```bash
docker service create \
  --name portainer-agent \
  --network portainer-agent-network \
  -e AGENT_CLUSTER_ADDR=tasks.portainer-agent \
  --mode global \
  --constraint 'node.platform.os == linux' \
  --mount type=bind,src=/var/run/docker.sock,dst=/var/run/docker.sock \
  --mount type=bind,src=/var/lib/docker/volumes,dst=/var/lib/docker/volumes \
  --publish mode=host,target=9001,published=9001 \
  portainer/agent:latest
```

Key flags:
- `--mode global` — runs on every node
- `--publish mode=host` — uses host networking, not mesh routing
- `AGENT_CLUSTER_ADDR` — enables agent cluster discovery

## Step 3: Create the Required Overlay Network

```bash
# Create an overlay network for Portainer agent communication
docker network create \
  --driver overlay \
  --attachable \
  portainer-agent-network

# Verify the network exists on all nodes
docker network ls | grep portainer-agent
docker network inspect portainer-agent-network
```

## Step 4: Full Portainer + Agent Swarm Stack

```yaml
# portainer-swarm.yml
version: "3.8"

services:
  # Portainer server — runs on a manager node
  portainer:
    image: portainer/portainer-ce:latest
    command: -H tcp://tasks.portainer-agent:9001 --tlsskipverify
    ports:
      - "9443:9443"
      - "9000:9000"
      - "8000:8000"
    volumes:
      - portainer_data:/data
    networks:
      - portainer-agent-network
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints:
          - node.role == manager

  # Portainer Agent — runs on every node
  portainer-agent:
    image: portainer/agent:latest
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /var/lib/docker/volumes:/var/lib/docker/volumes
    networks:
      - portainer-agent-network
    environment:
      AGENT_CLUSTER_ADDR: tasks.portainer-agent
    deploy:
      mode: global
      placement:
        constraints:
          - node.platform.os == linux

networks:
  portainer-agent-network:
    driver: overlay
    attachable: true

volumes:
  portainer_data:
```

Deploy with:

```bash
docker stack deploy -c portainer-swarm.yml portainer
```

## Step 5: Diagnose Overlay Network Issues

```bash
# Check if the overlay network is healthy
docker network inspect portainer-agent-network

# Look at "Peers" section — should list all Swarm nodes
# If nodes are missing, they can't communicate

# Test connectivity between nodes
# From a container on node1, ping a container on node2
docker exec -it $(docker ps -q -f name=portainer-agent) ping tasks.portainer-agent
```

## Step 6: Fix Node Communication Issues

```bash
# Check required Swarm ports are open between nodes
# Port 2377 — Swarm management (TCP)
# Port 7946 — Node communication (TCP/UDP)
# Port 4789 — Overlay network traffic (UDP)

# On each node, verify these ports are accessible
sudo ufw allow 2377/tcp
sudo ufw allow 7946/tcp
sudo ufw allow 7946/udp
sudo ufw allow 4789/udp
sudo ufw allow 9001/tcp  # Portainer agent
```

## Step 7: Check Service Logs Across All Nodes

```bash
# View logs for the agent service across all nodes
docker service logs portainer_portainer-agent

# View logs for specific node
docker service logs portainer_portainer-agent 2>&1 | grep <node-id>

# Follow logs in real time
docker service logs -f portainer_portainer-agent
```

## Step 8: Fix AGENT_CLUSTER_ADDR Issues

The `AGENT_CLUSTER_ADDR` variable tells each agent where to find its peers:

```bash
# Wrong: using a fixed IP (breaks when containers restart)
# AGENT_CLUSTER_ADDR=192.168.1.100:9001

# Correct: using the service's DNS name in Swarm
# AGENT_CLUSTER_ADDR=tasks.portainer-agent

# If you deployed without this variable, update the service:
docker service update \
  --env-add AGENT_CLUSTER_ADDR=tasks.portainer-agent \
  portainer_portainer-agent
```

## Step 9: Verify Agent Cluster Formation

```bash
# Check agent logs for cluster membership messages
docker service logs portainer_portainer-agent 2>&1 | grep -i "cluster\|member\|join"

# Look for messages like:
# "Cluster member detected: 10.0.1.2"
# If you only see one member, agents aren't discovering each other
```

## Step 10: Force Restart Agent Service

```bash
# Force restart the agent service to re-initialize cluster discovery
docker service update --force portainer_portainer-agent

# Check service tasks
docker service ps portainer_portainer-agent
```

## Conclusion

Agent communication issues on Docker Swarm are almost always caused by missing overlay network configuration, using mesh routing instead of `mode=host` for the agent port, or the `AGENT_CLUSTER_ADDR` variable pointing to the wrong address. The full stack deployment with the overlay network and global service mode is the most reliable approach and avoids most of these pitfalls.
