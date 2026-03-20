# How to Set Up Docker Swarm Overlay Networking with Custom IPv4 Subnets

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Docker, Docker Swarm, Overlay Networking, IPv4, Containers, Clustering

Description: Configure Docker Swarm overlay networks with custom IPv4 subnets for service communication and ingress routing, and customize the default address pools to avoid conflicts.

## Introduction

Docker Swarm uses two overlay networks by default: `ingress` (for published ports) and `docker_gwbridge` (for external connectivity). Customizing their subnets and creating custom overlay networks with specific subnets prevents IP conflicts in complex environments.

## Initializing Swarm with Custom Subnets

```bash
# Initialize Swarm with a custom advertise address
docker swarm init \
  --advertise-addr 192.168.1.10 \
  --default-addr-pool 10.50.0.0/16 \
  --default-addr-pool-mask-length 24
```

The `--default-addr-pool` sets the parent range; `--default-addr-pool-mask-length` sets the subnet size for each new overlay network.

## Customizing the ingress Network

The default `ingress` network uses `10.0.0.0/24`. To change it:

```bash
# Remove the existing ingress network
docker network rm ingress
# You will be warned that published services will be unavailable temporarily

# Recreate with a custom subnet
docker network create \
  --driver overlay \
  --ingress \
  --subnet 10.100.0.0/24 \
  --gateway 10.100.0.1 \
  ingress
```

## Creating Custom Overlay Networks per Stack

```bash
# Create overlay networks for different application tiers
docker network create \
  --driver overlay \
  --subnet 10.50.0.0/24 \
  --gateway 10.50.0.1 \
  frontend-overlay

docker network create \
  --driver overlay \
  --subnet 10.50.1.0/24 \
  --gateway 10.50.1.1 \
  backend-overlay
```

## Stack with Custom Overlay Networks

```yaml
# docker-stack.yml
version: "3.8"

services:
  web:
    image: nginx:alpine
    ports:
      - "80:80"
    networks:
      - frontend-overlay
    deploy:
      replicas: 3

  api:
    image: my-api:latest
    networks:
      - frontend-overlay
      - backend-overlay
    deploy:
      replicas: 2

  db:
    image: postgres:15-alpine
    networks:
      - backend-overlay
    deploy:
      replicas: 1
    environment:
      POSTGRES_PASSWORD: secret

networks:
  frontend-overlay:
    external: true
  backend-overlay:
    external: true
```

```bash
docker stack deploy -c docker-stack.yml myapp
```

## Verifying Overlay Network Connectivity

```bash
# Check services are running on all nodes
docker service ls
docker service ps web

# Test service resolution from within a container
docker exec -it $(docker ps -q --filter name=myapp_web) \
  nslookup api

# Check VXLAN tunnels
ip link show type vxlan
```

## Monitoring Overlay Network Traffic

```bash
# Capture VXLAN traffic on the host interface
sudo tcpdump -i eth0 -n "udp port 4789"
```

## Conclusion

Docker Swarm overlay networks use VXLAN for multi-host container communication. Customize default address pools during `docker swarm init`, recreate the ingress network for a clean subnet, and define per-stack overlay networks to achieve tier isolation in production Swarm deployments.
