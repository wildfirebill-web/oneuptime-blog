# How to Configure Docker Default Address Pool for IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Docker, IPv6, Address Pool, daemon.json, Network Subnets

Description: Configure Docker's default address pool to automatically allocate IPv6 subnets when creating new networks, preventing subnet conflicts and ensuring predictable IPv6 addressing across Docker hosts.

## Introduction

The `default-address-pools` setting in Docker's `daemon.json` defines the IPv6 (and IPv4) CIDR ranges from which Docker automatically allocates subnets when you create a new network without specifying a subnet. Each new network gets a `/size` subnet carved from the pool's base range. Configuring this prevents Docker from using conflicting default ranges and allows predictable subnet allocation across multiple Docker hosts.

## Configure Default Address Pool

```json
// /etc/docker/daemon.json
{
  "ipv6": true,
  "ip6tables": true,
  "default-address-pools": [
    {
      "base": "172.30.0.0/16",
      "size": 24
    },
    {
      "base": "fd00:docker::/48",
      "size": 64
    }
  ]
}
```

```bash
# Apply the configuration

sudo systemctl restart docker

# Verify the pool is configured
docker info | grep -A10 "Default Address Pools"
```

## How Docker Allocates from the Pool

```bash
# Create networks WITHOUT specifying subnets
# Docker automatically carves /64 from fd00:docker::/48

docker network create --ipv6 net1
docker network create --ipv6 net2
docker network create --ipv6 net3

# Each gets a unique subnet
docker network inspect net1 --format "{{range .IPAM.Config}}{{.Subnet}} {{end}}"
# fd00:docker:0:1::/64

docker network inspect net2 --format "{{range .IPAM.Config}}{{.Subnet}} {{end}}"
# fd00:docker:0:2::/64

docker network inspect net3 --format "{{range .IPAM.Config}}{{.Subnet}} {{end}}"
# fd00:docker:0:3::/64

# Clean up
docker network rm net1 net2 net3
```

## Multi-Host Configuration

```json
// Host A: /etc/docker/daemon.json
{
  "ipv6": true,
  "ip6tables": true,
  "default-address-pools": [
    {"base": "172.30.0.0/16", "size": 24},
    {"base": "fd00:docker:a::/48", "size": 64}
  ]
}

// Host B: /etc/docker/daemon.json
{
  "ipv6": true,
  "ip6tables": true,
  "default-address-pools": [
    {"base": "172.31.0.0/16", "size": 24},
    {"base": "fd00:docker:b::/48", "size": 64}
  ]
}
```

```bash
# With this setup, networks on Host A get fd00:docker:a::/64 subnets
# Networks on Host B get fd00:docker:b::/64 subnets
# No IPv6 conflicts when containers need to communicate across hosts
```

## Verify Pool Exhaustion

```bash
# Docker will fail with "could not find an available" error if pool is exhausted
# Check how many networks exist
docker network ls | wc -l

# Count available subnets in pool
# /48 base with /64 size = 2^16 = 65536 subnets per pool

# View all network subnets
docker network ls -q | xargs -I{} docker network inspect {} \
    --format "{{.Name}}: {{range .IPAM.Config}}{{.Subnet}} {{end}}" 2>/dev/null | \
    grep -v "^$"
```

## Conclusion

Configure `default-address-pools` in `daemon.json` with an IPv6 base CIDR and size to control automatic subnet allocation. A `/48` base with `/64` size creates 65,536 unique subnets for Docker networks. Assign different IPv6 pool ranges to different Docker hosts to prevent cross-host subnet conflicts. After configuration, new networks created without explicit subnets automatically receive IPv6 ranges from the pool. Restart Docker after any `daemon.json` changes.
