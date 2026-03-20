# How to Connect a Running Container to an Additional Docker IPv4 Network

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Docker, Networking, IPv4, docker network connect, Containers

Description: Add a running Docker container to an additional network using docker network connect, assign a static IP, and disconnect it when no longer needed without restarting the container.

## Introduction

`docker network connect` adds a running container to an additional network without interrupting it. This is useful for connecting a container to a management network, linking it to another application's network, or temporarily giving it access to a specific subnet.

## Connecting a Running Container to Another Network

```bash
# The container "web-app" is already running on "frontend-net"
# Connect it to "backend-net" as well
docker network connect backend-net web-app

# Verify — container now has two network interfaces
docker inspect web-app \
  --format '{{range $net, $cfg := .NetworkSettings.Networks}}{{$net}}: {{$cfg.IPAddress}}{{"\n"}}{{end}}'
```

Output:

```
frontend-net: 172.20.0.2
backend-net: 172.21.0.3
```

## Connecting with a Static IP

```bash
# Assign a specific IP on the new network
docker network connect --ip 10.10.0.100 management-net web-app
```

## Connecting with Aliases

Aliases let other containers on the network resolve this container by an alternative name:

```bash
# Other containers on backend-net can reach this as "web" or "web-service"
docker network connect --alias web --alias web-service backend-net web-app
```

## Listing a Container's Current Networks

```bash
docker inspect web-app --format '{{json .NetworkSettings.Networks}}' | python3 -m json.tool
```

## Disconnecting from a Network

```bash
# Remove the container from backend-net (container keeps running on other networks)
docker network disconnect backend-net web-app

# Force disconnect (ignores errors)
docker network disconnect -f backend-net web-app
```

## Practical Use Case: Connecting to a Monitoring Network

Add a production container to a monitoring network so Prometheus can scrape it:

```bash
# Create a monitoring network
docker network create monitoring --subnet 10.99.0.0/24

# Connect each service to the monitoring network
docker network connect --alias app-metrics monitoring my-app
docker network connect --alias db-metrics  monitoring my-postgres

# Now Prometheus can reach app-metrics:9090 and db-metrics:9187
```

## Connecting via Docker Compose

In Docker Compose, define additional networks and attach services:

```yaml
services:
  web:
    image: nginx:alpine
    networks:
      - frontend
      - monitoring

networks:
  frontend:
    external: true
  monitoring:
    external: true
```

## Conclusion

`docker network connect` is the live, zero-downtime way to add network connectivity to a running container. Pair it with `--ip` for static assignment and `--alias` for name-based discovery. Use `docker network disconnect` to remove connectivity cleanly.
