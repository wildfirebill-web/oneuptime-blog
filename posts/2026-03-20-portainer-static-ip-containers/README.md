# How to Configure Static IP Addresses for Containers in Portainer - Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Networking, Static IP, DevOps

Description: Learn how to assign fixed static IP addresses to Docker containers in Portainer using custom networks with defined subnets.

## Introduction

By default, Docker assigns container IP addresses dynamically from a network's subnet pool. While this works for most applications, some use cases require predictable, fixed IPs: legacy applications that hardcode connection strings, firewall rules that target specific container IPs, or monitoring configurations that reference fixed endpoints. Static IPs in Docker require a custom network with a defined subnet - the default bridge network does not support static IP assignment.

## Prerequisites

- Portainer installed with a connected Docker environment
- A custom Docker network (not the default bridge)

## Why Static IPs Require Custom Networks

The default `bridge` network (`docker0`) manages its address pool automatically and does not accept `--ip` assignments. You must create a custom network with an explicit subnet, then assign IPs within that subnet.

## Step 1: Create a Network with a Defined Subnet

```bash
# Create a custom bridge network with a specific subnet:

docker network create \
  --driver bridge \
  --subnet 172.25.0.0/24 \
  --gateway 172.25.0.1 \
  --ip-range 172.25.0.0/25 \   # Containers get IPs from .0 to .127
  static-ip-network

# Verify:
docker network inspect static-ip-network | jq '.[].IPAM'
```

Via Portainer:
1. Navigate to **Networks** → **Add network**.
2. Set Driver to `bridge`.
3. Set Subnet: `172.25.0.0/24`, Gateway: `172.25.0.1`.
4. Click **Create the network**.

## Step 2: Assign Static IP at Container Creation (CLI)

```bash
# Run a container with a fixed IP:
docker run -d \
  --name dns-server \
  --network static-ip-network \
  --ip 172.25.0.10 \          # Fixed IP within the subnet
  coredns/coredns:latest

# Run another container with a different fixed IP:
docker run -d \
  --name monitoring \
  --network static-ip-network \
  --ip 172.25.0.11 \
  prom/prometheus:latest

# Verify IP assignments:
docker inspect dns-server --format '{{.NetworkSettings.Networks.static-ip-network.IPAddress}}'
# 172.25.0.10
```

## Step 3: Assign Static IP via Portainer UI

1. Navigate to **Containers** → **Add container**.
2. Set the image name.
3. Under **Network**, select your custom network (`static-ip-network`).
4. Expand **Advanced network settings**.
5. Enter the IP address in the **IPv4 Address** field: `172.25.0.10`.
6. Click **Deploy the container**.

## Step 4: Static IPs in Docker Compose

```yaml
# docker-compose.yml with static IP assignments
version: "3.8"

services:
  # DNS server - always at .10
  coredns:
    image: coredns/coredns:latest
    restart: unless-stopped
    networks:
      infra-net:
        ipv4_address: 172.25.0.10   # Fixed: other containers use this as DNS

  # Prometheus - always at .20
  prometheus:
    image: prom/prometheus:latest
    restart: unless-stopped
    networks:
      infra-net:
        ipv4_address: 172.25.0.20   # Fixed: Grafana points here

  # Grafana - always at .21
  grafana:
    image: grafana/grafana:latest
    restart: unless-stopped
    ports:
      - "3000:3000"
    networks:
      infra-net:
        ipv4_address: 172.25.0.21   # Fixed: known address
    environment:
      - GF_SERVER_HTTP_PORT=3000

networks:
  infra-net:
    driver: bridge
    ipam:
      config:
        - subnet: 172.25.0.0/24
          gateway: 172.25.0.1
```

## Step 5: Reserve IPs to Avoid Conflicts

Divide the subnet to avoid DHCP/static conflicts:

```bash
172.25.0.0/24 layout:
  172.25.0.1        → Gateway
  172.25.0.2-9      → Reserved
  172.25.0.10-49    → Static assignments (reserved for named containers)
  172.25.0.50-200   → Dynamic pool (Docker assigns from here)
  172.25.0.201-254  → Reserved for host tools
```

Configure Docker to use only the dynamic pool for auto-assignment:

```bash
docker network create \
  --driver bridge \
  --subnet 172.25.0.0/24 \
  --gateway 172.25.0.1 \
  --ip-range 172.25.0.50/27 \   # Auto-assign only from .50 to .81
  static-ip-network
```

## Step 6: Verify Static IP After Restart

Static IPs persist through container restarts (stop/start), but are not preserved if the container is removed and recreated with a new name:

```bash
# Restart and verify IP is retained:
docker restart dns-server
docker inspect dns-server --format '{{.NetworkSettings.Networks.static-ip-network.IPAddress}}'
# Still: 172.25.0.10

# If the container is removed and recreated without --ip, it gets a dynamic IP:
docker rm dns-server
docker run -d --name dns-server --network static-ip-network coredns/coredns:latest
docker inspect dns-server --format '{{.NetworkSettings.Networks.static-ip-network.IPAddress}}'
# Dynamic IP now - must specify --ip again
```

Always define static IPs in your Compose file or deployment scripts so they are reproduced consistently.

## Conclusion

Static IP addresses for Docker containers require a custom network with an explicit subnet definition. Assign IPs using the `--ip` flag in CLI, the IPv4 Address field in Portainer's container creation form, or the `ipv4_address` key in Docker Compose network configuration. Reserve a portion of the subnet for static assignments and configure the `--ip-range` option to prevent Docker's automatic assignment from conflicting with your reserved addresses.
