# How to Assign Static IPv6 Addresses to Docker Containers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Docker, IPv6, Static IP, Container Networking, Fixed Address

Description: Assign fixed IPv6 addresses to Docker containers using --ip6 flag and Docker Compose ipv6_address option, ensuring predictable addressing for services requiring stable IPv6 identifiers.

## Introduction

Docker allows assigning static IPv6 addresses to containers when they connect to user-defined networks with IPv6 enabled. Static IPv6 assignment is useful for services that need stable addresses - such as databases, DNS servers, or internal services referenced by IPv6 address. The address must be within the network's configured IPv6 subnet range.

## Assign Static IPv6 with docker run

```bash
# First, create a network with IPv6 subnet

docker network create \
    --driver bridge \
    --ipv6 \
    --subnet 172.20.0.0/24 \
    --subnet fd00:static::/64 \
    --gateway 172.20.0.1 \
    --gateway fd00:static::1 \
    static-net

# Assign static IPv6 to a container
docker run -d \
    --name web \
    --network static-net \
    --ip6 fd00:static::10 \
    nginx:latest

# Assign both static IPv4 and IPv6
docker run -d \
    --name db \
    --network static-net \
    --ip 172.20.0.20 \
    --ip6 fd00:static::20 \
    postgres:15

# Verify addresses
docker inspect web --format "{{.NetworkSettings.Networks.static-net.GlobalIPv6Address}}"
# Output: fd00:static::10

docker inspect db --format "{{.NetworkSettings.Networks.static-net.GlobalIPv6Address}}"
# Output: fd00:static::20
```

## Static IPv6 in Docker Compose

```yaml
# compose.yaml

networks:
  appnet:
    driver: bridge
    enable_ipv6: true
    ipam:
      config:
        - subnet: 172.20.0.0/24
          gateway: 172.20.0.1
        - subnet: fd00:static::/64
          gateway: fd00:static::1

services:
  nginx:
    image: nginx:latest
    networks:
      appnet:
        ipv4_address: 172.20.0.10
        ipv6_address: fd00:static::10
    ports:
      - "80:80"

  redis:
    image: redis:7
    networks:
      appnet:
        ipv4_address: 172.20.0.20
        ipv6_address: fd00:static::20

  app:
    image: myapp:latest
    networks:
      - appnet
    environment:
      - REDIS_HOST=fd00:static::20
      - NGINX_HOST=fd00:static::10
```

## Test Static IPv6 Connectivity

```bash
# Start services
docker compose up -d

# Verify static addresses
docker compose exec nginx ip -6 addr show eth0
# Should show: fd00:static::10/64

docker compose exec redis ip -6 addr show eth0
# Should show: fd00:static::20/64

# Test connectivity using static IPv6
docker compose exec app ping6 -c 3 fd00:static::20

# Connect to Redis using static IPv6
docker compose exec app redis-cli -h fd00:static::20 ping
# Output: PONG
```

## Limitations and Considerations

```bash
# Static IPv6 ONLY works on user-defined networks
# The default bridge network does NOT support static IPv6 assignment

# Verify network type
docker network inspect static-net | grep '"Driver"'
# Must show: "Driver": "bridge" (user-defined)

# The --ip6 address must be within the subnet
# This will FAIL if address is outside the subnet:
docker run --rm \
    --network static-net \
    --ip6 fd00:other::99 \  # Wrong subnet! Will fail
    alpine echo "test"

# Always reserve .1 address for gateway
# Reserve range for Docker (auto-assigned) vs static
# Example: fd00:static::1 = gateway
#          fd00:static::2 to ::9 = Docker auto-assigned
#          fd00:static::10 to ::99 = manually assigned
```

## Conclusion

Assign static IPv6 addresses to Docker containers with `--ip6 <address>` in `docker run` or `ipv6_address` in Docker Compose network config. The address must be within the network's configured IPv6 subnet and the network must be a user-defined bridge (not the default bridge). Reserve a range within your subnet for static assignments to avoid conflicts with Docker's auto-assigned addresses. Static IPv6 addresses are useful for services referenced by address in configuration files or environment variables.
