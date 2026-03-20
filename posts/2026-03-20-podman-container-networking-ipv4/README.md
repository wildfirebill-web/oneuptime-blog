# How to Configure Podman Container Networking with IPv4 Subnets

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Networking, IPv4, Containers, CNI, Linux

Description: Configure custom IPv4 subnets for Podman containers using Podman networks to isolate workloads and control address allocation.

## Introduction

Podman is a daemonless container engine compatible with Docker's CLI. It uses CNI (Container Network Interface) or Netavark for networking. You can create custom networks with specific IPv4 subnets to organize and isolate container workloads, much like Docker's user-defined networks.

## Creating a Custom IPv4 Network

```bash
# Create a Podman network with a custom IPv4 subnet
podman network create \
  --subnet 10.89.0.0/24 \
  --gateway 10.89.0.1 \
  --driver bridge \
  myapp-net
```

## Listing and Inspecting Networks

```bash
# List all Podman networks
podman network ls

# Inspect network details including subnet and gateway
podman network inspect myapp-net
```

## Running Containers on a Custom Network

```bash
# Run a container on the custom network with a static IP
podman run -d \
  --name db \
  --network myapp-net \
  --ip 10.89.0.10 \       # Static IPv4 within the subnet
  -e POSTGRES_PASSWORD=secret \
  postgres:15

# Run an app container that connects to the database by name
podman run -d \
  --name api \
  --network myapp-net \
  --ip 10.89.0.11 \
  -e DB_HOST=db \          # Podman DNS resolves 'db' to 10.89.0.10
  myapp:latest
```

## Using Podman with Pods

Podman supports Kubernetes-style pods where containers share a network namespace:

```bash
# Create a pod with a defined subnet
podman pod create \
  --name mypod \
  --network myapp-net \
  --ip 10.89.0.20

# Add containers to the pod (they share the pod's IP)
podman run -d \
  --pod mypod \
  --name frontend \
  nginx

podman run -d \
  --pod mypod \
  --name backend \
  myapp:latest
```

## Podman Compose with Custom Networks

```yaml
# podman-compose.yml (compatible with docker-compose)
version: "3.8"

services:
  db:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: secret
    networks:
      appnet:
        ipv4_address: 10.89.0.10

  api:
    image: myapp:latest
    environment:
      DB_HOST: db
    networks:
      appnet:
        ipv4_address: 10.89.0.11

networks:
  appnet:
    driver: bridge
    ipam:
      config:
        - subnet: 10.89.0.0/24
          gateway: 10.89.0.1
```

## Rootless Networking

Podman supports rootless containers. Rootless networks use `slirp4netns` or `pasta` instead of CNI:

```bash
# Create a rootless network (no root required)
podman network create --subnet 10.90.0.0/24 rootless-net

# Run as a non-root user
podman run --network rootless-net --ip 10.90.0.5 nginx
```

## Verifying Network Connectivity

```bash
# Check container IP addresses
podman inspect db --format '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}'

# Test DNS resolution inside a container
podman exec -it api ping db
```

## Conclusion

Podman provides flexible IPv4 networking through CNI or Netavark with full support for custom subnets, static IPs, and pod-level network sharing. It is a drop-in alternative to Docker with enhanced security through rootless operation.
