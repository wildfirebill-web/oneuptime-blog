# How to Set Up Docker Swarm High Availability with Portainer - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker Swarm, High Availability, Infrastructure, Clustering

Description: Configure a highly available Docker Swarm cluster with multiple manager nodes and deploy Portainer to manage it with HA.

## Introduction

Docker Swarm achieves high availability through redundant manager nodes using the Raft consensus algorithm. With 3 or 5 manager nodes, the Swarm cluster tolerates manager failures while Portainer continues to manage it. This guide covers setting up a production-ready HA Swarm cluster.

## HA Architecture

- **3 manager nodes**: Tolerates 1 manager failure (quorum = 2)
- **5 manager nodes**: Tolerates 2 manager failures (quorum = 3)
- **N worker nodes**: No limit, used only for running workloads

## Step 1: Initialize the First Manager

```bash
# On manager1 (192.168.1.10)

docker swarm init \
  --advertise-addr 192.168.1.10 \
  --listen-addr 192.168.1.10:2377

# Save the manager join token
MANAGER_TOKEN=$(docker swarm join-token manager -q)
echo "Manager token: $MANAGER_TOKEN"

# Save the worker join token
WORKER_TOKEN=$(docker swarm join-token worker -q)
echo "Worker token: $WORKER_TOKEN"
```

## Step 2: Add Additional Manager Nodes

```bash
# On manager2 (192.168.1.11)
docker swarm join \
  --token $MANAGER_TOKEN \
  192.168.1.10:2377 \
  --advertise-addr 192.168.1.11

# On manager3 (192.168.1.12)
docker swarm join \
  --token $MANAGER_TOKEN \
  192.168.1.10:2377 \
  --advertise-addr 192.168.1.12

# Add worker nodes
# On worker1, worker2, worker3...
docker swarm join \
  --token $WORKER_TOKEN \
  192.168.1.10:2377
```

## Step 3: Verify the Swarm

```bash
# On any manager: verify cluster state
docker node ls
# ID               HOSTNAME    STATUS    AVAILABILITY  MANAGER STATUS   ENGINE VERSION
# xxx  *  manager1    Ready     Active      Leader
# yyy     manager2    Ready     Active      Reachable
# zzz     manager3    Ready     Active      Reachable
# aaa     worker1     Ready     Active

# Check Raft consensus status
docker node inspect manager1 --format '{{.ManagerStatus}}'
```

## Step 4: Deploy Portainer in HA Mode

For HA Portainer, use the Portainer agent + Swarm service:

```yaml
# portainer-swarm-stack.yml
version: '3.8'

services:
  agent:
    image: portainer/agent:latest
    environment:
      AGENT_CLUSTER_ADDR: tasks.agent
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /var/lib/docker/volumes:/var/lib/docker/volumes
    networks:
      - agent_network
    deploy:
      mode: global       # Run on ALL nodes
      placement:
        constraints: [node.platform.os == linux]

  portainer:
    image: portainer/portainer-ce:latest
    command: -H tcp://tasks.agent:9001 --tlsskipverify
    ports:
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
        constraints: [node.role == manager]
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3

networks:
  agent_network:
    driver: overlay
    attachable: true

volumes:
  portainer_data:
    driver: local
```

```bash
# Deploy the Portainer stack
docker stack deploy -c portainer-swarm-stack.yml portainer
```

## Step 5: Configure a Load Balancer for Managers

```nginx
# /etc/nginx/conf.d/swarm-lb.conf
upstream swarm_managers {
    server 192.168.1.10:9000;
    server 192.168.1.11:9000;
    server 192.168.1.12:9000;
}

server {
    listen 9000;
    location / {
        proxy_pass http://swarm_managers;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

## Step 6: Prevent Workloads on Manager Nodes

```bash
# Drain managers so they only run Swarm management tasks
docker node update --availability drain manager1
docker node update --availability drain manager2
docker node update --availability drain manager3

# Verify
docker node ls
# Managers should show AVAILABILITY = Drain
```

## Step 7: Test HA Failover

```bash
# Simulate a manager failure
# On manager1: stop Docker
sudo systemctl stop docker

# From manager2: verify cluster is still operational
docker node ls  # Should show manager1 as "Down"
docker service ls  # Services should still be running

# Restart manager1
sudo systemctl start docker

# Verify it rejoins
docker node ls  # manager1 should return as "Ready"
```

## Conclusion

A 3-manager Docker Swarm cluster with Portainer deployed as a Swarm service provides true high availability. The Raft consensus algorithm ensures the cluster remains operational even if one manager fails. Portainer's agent mode allows it to connect to the Swarm via the distributed agent network rather than a single Docker socket, enabling resilient management.
