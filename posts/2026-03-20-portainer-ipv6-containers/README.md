# How to Set Up IPv6 for Containers in Portainer - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, IPv6, Docker Networking, Network Configuration, Container Networking

Description: Learn how to enable and configure IPv6 for Docker containers in Portainer, including global daemon settings and per-network IPv6 configuration.

---

Docker supports IPv6 for containers, but it requires explicit configuration at both the daemon level and the network level. This guide covers enabling IPv6 globally and configuring IPv6 networks in Portainer stacks.

## Step 1: Enable IPv6 in Docker Daemon

Edit `/etc/docker/daemon.json` on the host:

```json
{
  "ipv6": true,
  "fixed-cidr-v6": "fd00::/64",
  "ip6tables": true
}
```

Restart Docker after changing daemon config:

```bash
sudo systemctl restart docker
```

The `fixed-cidr-v6` range is a ULA (Unique Local Address) prefix - suitable for internal container networking. Use a public IPv6 prefix if your host has one.

## Step 2: Create an IPv6-Enabled Network

Create a network with an IPv6 subnet:

```bash
docker network create \
  --driver bridge \
  --ipv6 \
  --subnet 172.28.0.0/16 \
  --subnet fd00:100::/80 \
  --gateway fd00:100::1 \
  ipv6_net
```

Or in Portainer's **Networks > Add network**:

- Enable **IPv6 Network**
- Set **IPv6 Subnet**: `fd00:100::/80`
- Set **IPv6 Gateway**: `fd00:100::1`

## Step 3: Use IPv6 Networks in a Stack

```yaml
version: "3.8"

services:
  api:
    image: my-api:latest
    networks:
      - ipv6_net
    environment:
      # Bind to all interfaces including IPv6
      LISTEN_ADDR: "0.0.0.0"

  web:
    image: nginx:alpine
    networks:
      - ipv6_net
    ports:
      - "80:80"
      - "443:443"

networks:
  ipv6_net:
    driver: bridge
    enable_ipv6: true
    ipam:
      config:
        - subnet: 172.28.0.0/16
        - subnet: fd00:100::/80
          gateway: fd00:100::1
```

## Verifying IPv6 Connectivity

```bash
# Check the container's IPv6 address

docker inspect $(docker ps -qf name=api) | \
  jq '.[0].NetworkSettings.Networks[].GlobalIPv6Address'

# Test IPv6 connectivity from inside the container
docker exec -it $(docker ps -qf name=api) ping6 fd00:100::1

# Test connectivity between containers using IPv6
docker exec -it $(docker ps -qf name=web) ping6 fd00:100::2
```

## Nginx Listening on IPv6

Configure Nginx to accept both IPv4 and IPv6 connections:

```nginx
server {
    listen 80;
    listen [::]:80;         # IPv6 listener
    listen 443 ssl;
    listen [::]:443 ssl;    # IPv6 SSL listener

    server_name example.com;
    # ...
}
```

## IPv6 with Macvlan (Global IPv6 Addresses)

For containers that need globally routable IPv6 addresses from your ISP prefix:

```bash
# Assuming your ISP assigned 2001:db8::/48 to your router
# And your Docker host is on 2001:db8:0:1::/64

docker network create \
  --driver macvlan \
  --opt parent=eth0 \
  --subnet 2001:db8:0:1::/64 \
  --ipv6 \
  global_ipv6_net
```

## Common Issues

| Issue | Cause | Fix |
|-------|-------|-----|
| Container has no IPv6 address | IPv6 not enabled on network | Add `enable_ipv6: true` to network |
| Cannot ping IPv6 gateway | `ip6tables` not enabled | Set `"ip6tables": true` in daemon.json |
| IPv6 works internally but not externally | No IPv6 masquerading | Add `ip6tables -t nat -A POSTROUTING -s fd00::/64 -j MASQUERADE` |
| Containers lose IPv6 after restart | Routes not persisted | Add routes to systemd-networkd or netplan |
