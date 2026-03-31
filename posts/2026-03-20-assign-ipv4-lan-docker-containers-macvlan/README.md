# How to Assign IPv4 Addresses from Your LAN to Docker Containers with macvlan

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Docker, Networking, Macvlan, IPv4, LAN, Container

Description: Configure Docker macvlan networking so containers receive IPv4 addresses from the physical LAN subnet, making them reachable from any device on the network without port mapping.

## Introduction

Using LAN IPv4 addresses for Docker containers removes the need for port mapping and NAT, letting containers behave like physical servers. The key is coordinating the container IP range with your DHCP server to avoid conflicts.

## Planning the Address Space

Before configuring, reserve a range in your LAN DHCP server's exclusion list:

| LAN | DHCP Range | Docker Container Range |
|---|---|---|
| 192.168.1.0/24 | 192.168.1.10–.199 | 192.168.1.220–.250 |

The container range (192.168.1.220–.250) must be excluded from the DHCP pool to prevent conflicts.

## Creating the macvlan Network

```bash
# Use the full LAN subnet, gateway, and restrict auto-assigned IPs

docker network create \
  --driver macvlan \
  --subnet 192.168.1.0/24 \
  --gateway 192.168.1.1 \
  --ip-range 192.168.1.220/27 \
  --opt parent=eth0 \
  lan-containers
```

`192.168.1.220/27` covers `.220–.251` - a clean 32-address block.

## Assigning LAN IPs to Containers

```bash
# Container with auto-assigned IP from the range
docker run -d \
  --name homeassistant \
  --network lan-containers \
  --restart unless-stopped \
  ghcr.io/home-assistant/home-assistant:stable

# Container with specific LAN IP
docker run -d \
  --name pihole \
  --network lan-containers \
  --ip 192.168.1.220 \
  --restart unless-stopped \
  -e WEBPASSWORD=admin \
  pihole/pihole:latest
```

## Verifying LAN Reachability

```bash
# From a laptop on the same LAN
ping 192.168.1.220   # Should reach the pihole container

# From another host
curl http://192.168.1.220/admin
```

## Running Multiple Services on LAN IPs

```yaml
# docker-compose.yml
version: "3.8"

services:
  pihole:
    image: pihole/pihole:latest
    networks:
      lan:
        ipv4_address: 192.168.1.220

  homeassistant:
    image: ghcr.io/home-assistant/home-assistant:stable
    networks:
      lan:
        ipv4_address: 192.168.1.221

networks:
  lan:
    driver: macvlan
    driver_opts:
      parent: eth0
    ipam:
      config:
        - subnet: 192.168.1.0/24
          gateway: 192.168.1.1
          ip_range: 192.168.1.220/27
```

## Host-to-Container Communication (Important)

By default, macvlan isolates the Docker host from its containers. The host cannot ping or reach containers on the macvlan network. To fix this, see the related article on macvlan bridge mode / macvlan host communication.

## Registering Container IPs in DHCP/DNS

For services to be findable by name, add DNS entries on your router or internal DNS:

```text
pihole.local → 192.168.1.220
homeassistant.local → 192.168.1.221
```

## Conclusion

macvlan with LAN IPs is the cleanest way to expose Docker services without port mapping. Reserve the container IP range in your DHCP server, create the network with the correct `--ip-range`, and assign fixed IPs to services that need stable addresses.
