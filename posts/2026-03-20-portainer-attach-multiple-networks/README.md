# How to Attach Containers to Multiple Networks in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Networking, Multi-Network, DevOps

Description: Learn how to attach Docker containers to multiple networks in Portainer to enable controlled communication between isolated network segments.

## Introduction

Attaching containers to multiple networks is a core pattern for multi-tier application architectures. A container can belong to several Docker networks simultaneously, enabling it to act as a bridge between network segments — for example, an API server that is reachable from a frontend network while also accessing a private database network. Portainer makes this straightforward both at container creation and for running containers.

## Prerequisites

- Portainer installed with a connected Docker environment
- Two or more Docker networks already created

## Why Multiple Networks

```
Frontend Network (dmz)    ←→    nginx (proxy)    ←→    App Network (backend)
                                     ↓
                          app (api server)       ←→    DB Network (data)
                                                            ↓
                                                       postgres (db)
```

- `nginx` is on `dmz` and `backend` — receives external traffic, proxies to `app`
- `app` is on `backend` and `data` — talks to both nginx and postgres
- `postgres` is only on `data` — not reachable from dmz

## Step 1: Create Networks First

```bash
# Create three isolated networks:
docker network create --driver bridge dmz-network
docker network create --driver bridge --internal app-network
docker network create --driver bridge --internal data-network

# Verify:
docker network ls | grep -E "dmz|app|data"
```

## Step 2: Attach Multiple Networks at Container Creation (Portainer UI)

1. Navigate to **Containers** in Portainer.
2. Click **Add container**.
3. Scroll to the **Network** section.
4. Select the first network from the dropdown and click **+Add network**.
5. Repeat to add additional networks.
6. Click **Deploy the container**.

Note: Portainer may only show one network selection in the basic view. Deploy with one network, then connect additional networks post-creation (Step 3).

## Step 3: Connect a Running Container to Additional Networks

### Via Portainer UI

1. Navigate to the container's detail page.
2. Scroll to **Connected networks**.
3. Click the network dropdown and select the network to join.
4. Click **Join network**.

### Via CLI

```bash
# Connect a running container to additional networks:
docker network connect app-network my-api-container
docker network connect data-network my-api-container

# Verify the container is on multiple networks:
docker inspect my-api-container --format '{{range $name, $conf := .NetworkSettings.Networks}}{{$name}} {{end}}'
# Output: dmz-network app-network data-network
```

## Step 4: Multi-Network Docker Compose Configuration

```yaml
# docker-compose.yml — multi-tier architecture
version: "3.8"

services:
  nginx:
    image: nginx:alpine
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    networks:
      - dmz        # Receives external traffic
      - backend    # Proxies to app containers
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro

  api:
    image: myorg/api:latest
    restart: unless-stopped
    networks:
      - backend    # Receives proxied requests from nginx
      - data       # Accesses databases
    environment:
      - DATABASE_URL=postgresql://user:pass@postgres:5432/mydb
      - REDIS_URL=redis://redis:6379

  postgres:
    image: postgres:15-alpine
    restart: unless-stopped
    networks:
      - data       # Only accessible from data network
    environment:
      - POSTGRES_PASSWORD=${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    networks:
      - data       # Only accessible from data network

# Network definitions
networks:
  dmz:
    driver: bridge

  backend:
    driver: bridge
    internal: true   # No direct internet access

  data:
    driver: bridge
    internal: true   # No direct internet access

volumes:
  postgres_data:
```

## Step 5: Assign Static IPs Per Network

When a container is on multiple networks, you can assign a specific IP on each:

```yaml
services:
  api:
    image: myorg/api:latest
    networks:
      backend:
        ipv4_address: 172.20.1.10   # Fixed IP on backend network
      data:
        ipv4_address: 172.21.1.10   # Fixed IP on data network

networks:
  backend:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.1.0/24

  data:
    driver: bridge
    ipam:
      config:
        - subnet: 172.21.1.0/24
```

## Step 6: Disconnect a Container from a Network

```bash
# Disconnect from a network (container continues running on other networks):
docker network disconnect app-network my-container

# Via Portainer UI:
# 1. Container detail page → Connected networks
# 2. Click Leave next to the network to remove
```

## Step 7: Inspect Network Membership

```bash
# Show all networks a container is connected to:
docker inspect my-api-container | jq '.[].NetworkSettings.Networks | keys'

# Show all containers on a specific network:
docker network inspect data-network | jq '.[].Containers | to_entries[] | {name: .value.Name, ip: .value.IPv4Address}'
```

## Conclusion

Attaching containers to multiple networks is the standard pattern for implementing network segmentation in Docker. An nginx proxy joins both the public-facing DMZ and the internal backend network, while database containers remain isolated on a data-only network. Portainer supports this through the UI's connected networks section and the Join network action on running containers. Combined with Docker Compose's network definitions, this pattern provides defense-in-depth for containerized applications without complex firewall rules.
