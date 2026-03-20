# How to Configure Docker Swarm IPv6 Overlay Networks

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Docker Swarm, IPv6, Overlay Network, Container Networking, Docker

Description: A guide to configuring Docker Swarm overlay networks with IPv6 support, enabling IPv6 connectivity for services in a Docker Swarm cluster.

Docker Swarm's overlay networking supports IPv6 through dual-stack configuration. Services in the swarm can communicate over IPv6 overlay addresses, and external clients can reach services via IPv6 if the ingress network is configured for dual-stack.

## Prerequisites

```bash
# Enable IPv6 in Docker daemon

cat > /etc/docker/daemon.json << 'EOF'
{
  "ipv6": true,
  "fixed-cidr-v6": "fd00:docker::/80",
  "experimental": true
}
EOF

sudo systemctl restart docker

# Verify IPv6 is enabled
docker info | grep IPv6
```

## Initialize Swarm

```bash
# Initialize swarm on manager node
docker swarm init --advertise-addr 192.168.1.10

# Or with IPv6 advertise address
docker swarm init --advertise-addr [2001:db8::manager]

# Join worker nodes
docker swarm join --token <token> 192.168.1.10:2377
```

## Create IPv6-Enabled Overlay Network

```bash
# Create dual-stack overlay network
docker network create \
  --driver overlay \
  --attachable \
  --ipam-driver default \
  --subnet 10.10.0.0/24 \
  --ipv6 \
  --subnet fd00:overlay::/64 \
  ipv6-overlay

# Verify network creation
docker network inspect ipv6-overlay | python3 -m json.tool | grep -A 5 "IPAM"
```

## Deploy Service on IPv6 Overlay

```bash
# Deploy a service on the IPv6 overlay network
docker service create \
  --name web \
  --network ipv6-overlay \
  --replicas 3 \
  --publish published=80,target=80 \
  nginx:alpine

# Verify service has IPv6 address
docker service inspect web --pretty

# Check container IPv6 addresses
docker ps | grep web
docker exec <container_id> ip -6 addr show
```

## Docker Stack with IPv6

```yaml
# docker-compose.yml (for docker stack deploy)

version: '3.9'

networks:
  ipv6-network:
    driver: overlay
    attachable: true
    enable_ipv6: true
    ipam:
      config:
        - subnet: 10.20.0.0/24
        - subnet: fd00:stack::/64

services:
  web:
    image: nginx:alpine
    networks:
      - ipv6-network
    ports:
      - "80:80"
      - "443:443"
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s

  api:
    image: my-api:latest
    networks:
      - ipv6-network
    deploy:
      replicas: 2
```

```bash
# Deploy the stack
docker stack deploy -c docker-compose.yml myapp

# Verify services
docker stack services myapp

# Check IPv6 connectivity between services
docker exec <web-container> ping6 api
```

## IPv6 Ingress Network

```bash
# The default ingress network needs IPv6 for external IPv6 access
# Recreate ingress with IPv6 (requires draining swarm)

# Remove existing ingress
docker network rm ingress
# WARNING: This removes load balancing temporarily

# Create new ingress with IPv6
docker network create \
  --driver overlay \
  --ingress \
  --subnet 10.0.0.0/24 \
  --ipv6 \
  --subnet fd00:ingress::/64 \
  ingress

# Verify
docker network inspect ingress | grep -A 10 "IPAM"
```

## Service Discovery Over IPv6

```bash
# Services in the same overlay network can communicate via service name
# Docker's embedded DNS resolves service names to both IPv4 and IPv6

# From within a container:
docker exec <container_id> nslookup web
# Returns both IPv4 (10.20.x.x) and IPv6 (fd00:stack::x) addresses

# Ping another service over IPv6
docker exec <container_id> ping6 api
```

## Troubleshooting IPv6 Overlay

```bash
# Check overlay network configuration
docker network inspect ipv6-overlay

# Verify containers have IPv6 addresses
docker exec <container_id> ip -6 addr show

# Check IPv6 routing in container
docker exec <container_id> ip -6 route show

# Test IPv6 connectivity between nodes
docker exec <container_id> ping6 fd00:overlay::another-container
```

Docker Swarm IPv6 overlay networking enables microservices to communicate over IPv6 within the cluster, with dual-stack configuration allowing gradual migration from IPv4-only to IPv6-capable services.
