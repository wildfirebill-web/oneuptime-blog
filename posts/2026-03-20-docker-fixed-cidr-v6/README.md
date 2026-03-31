# How to Configure fixed-cidr-v6 in Docker daemon.json

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Docker, IPv6, Fixed-cidr-v6, CIDR, Daemon, Networking

Description: Configure the fixed-cidr-v6 option in Docker's daemon.json to assign a specific IPv6 subnet to the default bridge network, understand CIDR sizing requirements, and verify address allocation.

## Introduction

The `fixed-cidr-v6` option in Docker's `daemon.json` specifies the IPv6 CIDR block assigned to the default bridge (`docker0`) network. Docker allocates IPv6 addresses to containers from this range. The prefix must be at least a `/64` for SLAAC to work correctly. Using a ULA prefix (`fd00::/8`) is recommended for private container networking. This setting applies only to the default bridge; custom networks can have their own subnets.

## Choose the Right CIDR Size

```bash
# Good: /80 allows up to 2^48 container addresses per host

# Fixed subnet for docker0 bridge:
# /80 = subnet of /48 parent, gives each host a unique /80 block

# daemon.json
# "fixed-cidr-v6": "fd00:dead:beef::/80"

# Minimum: /64 required for SLAAC
# "fixed-cidr-v6": "fd00:dead:beef::/64"

# Docker assigns /128 (individual) addresses to each container
# Example: fd00:dead:beef::2, fd00:dead:beef::3, etc.
```

## Configure fixed-cidr-v6

```json
// /etc/docker/daemon.json
{
  "ipv6": true,
  "fixed-cidr-v6": "fd00:dead:beef::/80",
  "ip6tables": true
}
```

```bash
sudo systemctl restart docker

# Verify the subnet is assigned to docker0
docker network inspect bridge | grep -A3 "v6"

# Sample output:
# "EnableIPv6": true,
# "IPAM": {
#   "Config": [
#     {"Subnet": "fd00:dead:beef::/80", "Gateway": "fd00:dead:beef::1"}
#   ]
# }
```

## Check Container IPv6 Address Allocation

```bash
# Run a container and inspect its IPv6
docker run -d --name test-ipv6 nginx

# Get container IPv6 address
docker inspect test-ipv6 | grep -A2 "GlobalIPv6Address"

# Or exec into container
docker exec test-ipv6 ip -6 addr show eth0
# Output: inet6 fd00:dead:beef::2/80 scope global

# Confirm gateway
docker exec test-ipv6 ip -6 route show default
# Output: default via fd00:dead:beef::1 dev eth0

# Clean up
docker rm -f test-ipv6
```

## Using Multiple Hosts with Different /80 Subnets

```bash
# For multi-host Docker setups, give each host a unique /80
# from the same /48 parent:

# Host 1: fd00:dead:beef:0:1::/80
# Host 2: fd00:dead:beef:0:2::/80
# Host 3: fd00:dead:beef:0:3::/80

# Host 1 daemon.json:
# "fixed-cidr-v6": "fd00:dead:beef:0:1::/80"

# Host 2 daemon.json:
# "fixed-cidr-v6": "fd00:dead:beef:0:2::/80"

# This avoids IP conflicts when containers on different hosts communicate
```

## Conclusion

The `fixed-cidr-v6` option in `daemon.json` sets the IPv6 subnet for the default Docker bridge network. Use a ULA prefix under `fd00::/8` with at least a `/64` prefix length. For multi-host environments, assign unique subnets (e.g., `/80`) to each Docker host from a shared `/48` parent prefix. After changing `fixed-cidr-v6`, restart Docker and verify with `docker network inspect bridge`. Containers automatically receive addresses from the configured range.
