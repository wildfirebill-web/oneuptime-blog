# How to Configure a Docker Bridge Network Subnet and Gateway

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Docker, Networking, IPv4, Bridge, Subnets, Gateway

Description: Configure the subnet, gateway, and IP allocation range for a Docker bridge network using docker network create and the IPAM driver options.

## Introduction

Docker bridge networks use IPAM (IP Address Management) to define the subnet and gateway. Properly configuring these ensures containers get predictable addresses, avoids conflicts with existing networks, and allows you to plan the IP space before deploying services.

## Creating a Bridge Network with Subnet and Gateway

```bash
# Basic subnet + gateway

docker network create \
  --driver bridge \
  --subnet 10.10.0.0/24 \
  --gateway 10.10.0.1 \
  prod-network
```

## Configuring Multiple IPAM Pools

You can specify multiple subnets in a single network (for advanced use cases):

```bash
docker network create \
  --driver bridge \
  --subnet 10.10.0.0/24 \
  --gateway 10.10.0.1 \
  --subnet 10.10.1.0/24 \
  --gateway 10.10.1.1 \
  dual-subnet-network
```

## Restricting the IP Allocation Range

`--ip-range` restricts which addresses Docker automatically assigns (while the full subnet is still routable):

```bash
# Subnet is /24, but auto-assign only from .100–.200
docker network create \
  --driver bridge \
  --subnet 10.10.0.0/24 \
  --gateway 10.10.0.1 \
  --ip-range 10.10.0.100/25 \
  controlled-network
```

Addresses outside the ip-range (.1–.99, .201–.254) can still be manually assigned with `--ip`.

## Disabling IPv6 Entirely

```bash
docker network create \
  --driver bridge \
  --subnet 10.10.0.0/24 \
  --gateway 10.10.0.1 \
  --opt "com.docker.network.bridge.enable_ipv6=false" \
  ipv4-only-network
```

## Inspecting Network Configuration

```bash
# Full network details including IPAM configuration
docker network inspect prod-network

# Show just the IPAM config
docker network inspect prod-network \
  --format '{{json .IPAM}}' | python3 -m json.tool
```

Output:

```json
{
  "Driver": "default",
  "Config": [
    {
      "Subnet": "10.10.0.0/24",
      "Gateway": "10.10.0.1",
      "IPRange": "10.10.0.100/25"
    }
  ]
}
```

## Docker Compose Network with Full IPAM Configuration

```yaml
networks:
  prod-net:
    driver: bridge
    ipam:
      driver: default
      config:
        - subnet: 10.10.0.0/24
          gateway: 10.10.0.1
          ip_range: 10.10.0.100/25
          aux_addresses:
            reserved-host: 10.10.0.5
```

The `aux_addresses` field reserves specific IPs for external use without allocating them to containers.

## Common Subnet Planning

| Environment | Suggested Range |
|---|---|
| Development | 172.20.0.0/24 |
| Staging | 172.21.0.0/24 |
| Production | 10.100.0.0/24 |
| Database tier | 10.100.1.0/24 |

## Conclusion

Define subnet and gateway at network creation time using `--subnet` and `--gateway`. Use `--ip-range` to partition the pool for auto-assignment while keeping the rest available for static assignment. Inspect with `docker network inspect` to verify IPAM configuration before deploying services.
