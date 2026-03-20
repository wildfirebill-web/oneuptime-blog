# How to Set Up Docker Compose Networking with Custom IPv4 Subnets

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Docker, Docker Compose, Networking, IPv4, Subnets, Containers

Description: Configure custom IPv4 subnets for Docker Compose services using the networks section with IPAM configuration, and connect multiple services to different network tiers.

## Introduction

Docker Compose automatically creates a network for each project, but the default subnet is random. Defining custom networks in the `networks:` section of your Compose file ensures consistent, predictable IP addressing and enables network-tier isolation.

## Basic Custom Network in Docker Compose

```yaml
# docker-compose.yml

version: "3.8"

services:
  web:
    image: nginx:alpine
    networks:
      - frontend

  app:
    image: my-app:latest
    networks:
      - frontend
      - backend

  db:
    image: postgres:15-alpine
    networks:
      - backend
    environment:
      POSTGRES_PASSWORD: secret

networks:
  frontend:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/24
          gateway: 172.20.0.1

  backend:
    driver: bridge
    ipam:
      config:
        - subnet: 172.21.0.0/24
          gateway: 172.21.0.1
```

In this configuration:
- `web` and `app` can communicate (both on `frontend`)
- `app` and `db` can communicate (both on `backend`)
- `web` cannot directly reach `db` (not on the same network)

## Running the Stack

```bash
docker compose up -d

# Verify network and container IPs
docker compose ps
docker network inspect $(docker compose config --format json | python3 -m json.tool | grep -o '"[^"]*_frontend"')
```

## Checking Service IPs

```bash
# Show IP addresses of running services
docker compose exec web hostname -I
docker compose exec app hostname -I
docker compose exec db hostname -I
```

## Using External Networks

To connect Compose services to an existing Docker network:

```yaml
networks:
  shared-net:
    external: true
    name: my-existing-network
```

## Naming the Project Network

By default, network names are prefixed with the project name. Set the project name:

```bash
# Use a specific project name to control network naming
COMPOSE_PROJECT_NAME=myapp docker compose up -d
# Creates network: myapp_frontend, myapp_backend
```

## Conclusion

Define networks in the `networks:` section with IPAM configuration to control subnets. Attach services to multiple networks for tier isolation - web servers on a frontend network, databases on a backend network, with application containers bridging both. This reflects real production network segmentation in a development environment.
