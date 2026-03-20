# How to Create a Docker Overlay Network for Multi-Host IPv4 Communication

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Docker, Networking, Overlay, IPv4, Multi-Host, Docker Swarm

Description: Create a Docker overlay network to enable IPv4 communication between containers running on different Docker hosts using Docker Swarm mode.

## Introduction

Docker overlay networks use VXLAN encapsulation to create a Layer 2 network spanning multiple Docker hosts. Containers on different physical machines communicate as if they were on the same local network. Overlay networks require Docker Swarm mode.

## Initializing Docker Swarm

```bash
# On the first host (manager)

docker swarm init --advertise-addr 192.168.1.10

# Output includes a join token, e.g.:
# docker swarm join --token SWMTKN-... 192.168.1.10:2377
```

```bash
# On worker hosts - run the join command from the output above
docker swarm join --token SWMTKN-... 192.168.1.10:2377
```

## Creating an Overlay Network

```bash
# On the Swarm manager
docker network create \
  --driver overlay \
  --subnet 10.20.0.0/24 \
  --gateway 10.20.0.1 \
  --attachable \
  my-overlay
```

The `--attachable` flag allows standalone containers (not just services) to attach to the network.

## Verifying the Network

```bash
docker network ls
docker network inspect my-overlay
```

## Deploying Services on the Overlay

```bash
# Create a Swarm service attached to the overlay
docker service create \
  --name web \
  --network my-overlay \
  --replicas 3 \
  nginx:alpine

# Check which nodes the replicas run on
docker service ps web
```

## Container-to-Container Communication Across Hosts

```bash
# Run a test container on the overlay from any Swarm node
docker run --rm --network my-overlay alpine ping -c 3 web
```

Docker's embedded DNS resolves `web` to the service's virtual IP (VIP), which load-balances across all replicas.

## Using Docker Compose with Overlay Networks

```yaml
# docker-compose.yml (deploy to Swarm with docker stack deploy)
version: "3.8"

services:
  web:
    image: nginx:alpine
    networks:
      - overlay-net

  api:
    image: my-api:latest
    networks:
      - overlay-net

networks:
  overlay-net:
    driver: overlay
    ipam:
      config:
        - subnet: 10.20.0.0/24
```

```bash
# Deploy the stack
docker stack deploy -c docker-compose.yml mystack
```

## Checking Overlay VXLAN Ports

Overlay networks require these ports open between hosts:

- TCP 2377 (Swarm management)
- TCP/UDP 7946 (node communication)
- UDP 4789 (VXLAN data plane)

```bash
# Check that VXLAN port is open
sudo ss -ulnp | grep 4789
```

## Conclusion

Docker overlay networks require Swarm mode and VXLAN ports between hosts. Create with `--driver overlay` and `--attachable` for flexibility, deploy services on the overlay, and benefit from Docker's built-in DNS load balancing across multi-host container deployments.
