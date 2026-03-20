# How to Configure Docker Networking for Containers with Overlapping IPv4 Subnets

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Docker, Networking, IPv4, Overlapping Subnets, Network Isolation

Description: Manage Docker containers with overlapping IPv4 subnets by using separate bridge networks for isolation, understanding routing behavior, and preventing cross-network communication between overlapping ranges.

## Introduction

Overlapping subnets in Docker occur when two networks share the same or overlapping IP ranges. While Docker isolates bridge networks so containers on different bridges cannot communicate directly, overlapping subnets can cause routing issues on the host. This guide explains how to safely manage this scenario.

## Why Overlapping Subnets Are Problematic

When two Docker bridge networks have overlapping subnets, the host routing table has two routes for the same CIDR. The kernel uses only one, causing packets destined for one network's containers to be misrouted to the other.

```bash
# Both networks use 172.20.0.0/24 — problematic
docker network create --subnet 172.20.0.0/24 network-a
docker network create --subnet 172.20.0.0/24 network-b  # conflict!

# Check the host routing table
ip route show | grep "172.20.0.0"
# Only one route exists — Docker created duplicate
```

## Prevention: Use Non-Overlapping Subnets

The best fix is using distinct subnets:

```bash
docker network create --subnet 172.20.0.0/24 network-a
docker network create --subnet 172.21.0.0/24 network-b
```

## Handling Legacy or External Overlapping Networks

When you must connect to an external network that overlaps with a Docker network:

```bash
# Rename the Docker network to use a different, safe range
docker network rm network-a
docker network create --subnet 10.200.0.0/24 network-a

# Verify no overlap with external network
ip route show
```

## Isolating Overlapping Networks with Network Namespaces

For advanced use cases where containers in separate namespaces intentionally have the same IP range (e.g., multi-tenant):

```bash
# Create two fully isolated networks with the same subnet
# They cannot communicate, so overlap is acceptable
docker network create \
  --subnet 192.168.100.0/24 \
  --opt "com.docker.network.bridge.enable_ip_masquerade=true" \
  tenant-a-network

docker network create \
  --subnet 192.168.100.0/24 \
  --opt "com.docker.network.bridge.enable_ip_masquerade=true" \
  tenant-b-network

# Verify both have separate bridge interfaces
ip link show type bridge | grep br-
```

The two bridges are isolated at Layer 2 — containers on `tenant-a-network` cannot reach `tenant-b-network` even with the same IP range.

## Checking for Subnet Conflicts

```bash
#!/bin/bash
# List all Docker network subnets to check for overlaps
docker network ls --format '{{.Name}}' | while read net; do
    subnet=$(docker network inspect "$net" \
      --format '{{range .IPAM.Config}}{{.Subnet}}{{end}}' 2>/dev/null)
    [ -n "$subnet" ] && echo "$net: $subnet"
done | sort -t. -k2,2n
```

## Using Different Address Pools Per Environment

```json
{
  "default-address-pools": [
    {"base": "10.200.0.0/16", "size": 24}
  ]
}
```

This prevents Docker from automatically picking subnets that conflict with your VPN or LAN.

## Conclusion

Prevent overlapping Docker subnets by using `default-address-pools` in `daemon.json` to control Docker's allocation range. For multi-tenant isolation where the same subnet must exist in multiple networks, Docker bridge isolation ensures containers cannot cross network boundaries even with identical IP ranges.
