# How to Configure Docker Bridge Networks with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Docker, IPv6, Bridge Network, docker0, Custom Bridge

Description: Configure Docker bridge networks for IPv6, understand the difference between the default bridge and user-defined bridges, set IPv6 options on bridges, and verify container IPv6 connectivity.

## Introduction

The Docker bridge driver is the default network driver and the most common for container networking. Docker creates a virtual bridge interface (like `docker0` for the default bridge) and connects containers through veth pairs. Bridges support IPv6 when `ipv6` is enabled in `daemon.json` and when subnets are specified. User-defined bridges offer better IPv6 support with DNS resolution and configurable subnets.

## Default Bridge with IPv6

```bash
# Enable IPv6 on default bridge via daemon.json
# /etc/docker/daemon.json:
# {
#   "ipv6": true,
#   "fixed-cidr-v6": "fd00:bridge::/80",
#   "ip6tables": true
# }

sudo systemctl restart docker

# Verify docker0 bridge has IPv6
ip -6 addr show docker0
# inet6 fd00:bridge::1/80 scope global

# Run container on default bridge
docker run -d --name test nginx

# Check container IPv6
docker exec test ip -6 addr show eth0
# inet6 fd00:bridge::2/80 scope global

# Note: default bridge containers cannot resolve each other by name
# Use user-defined bridges for DNS between containers
```

## User-Defined Bridge with IPv6

```bash
# Create user-defined bridge with explicit IPv6 subnet
docker network create \
    --driver bridge \
    --ipv6 \
    --subnet 172.18.0.0/24 \
    --subnet fd00:userbridge::/64 \
    --gateway 172.18.0.1 \
    --gateway fd00:userbridge::1 \
    mybridge

# Run containers on user-defined bridge
docker run -d --name web --network mybridge nginx
docker run -d --name api --network mybridge myapp

# DNS resolution works in user-defined bridges
docker exec api ping6 web  # Resolves 'web' to IPv6 address!

# Verify bridge interface
ip -6 addr show br-$(docker network inspect mybridge --format "{{.Id}}" | head -c 12)
```

## Bridge Options for IPv6

```bash
# Create bridge with custom options
docker network create \
    --driver bridge \
    --ipv6 \
    --subnet 172.19.0.0/24 \
    --subnet fd00:optbridge::/64 \
    --opt com.docker.network.bridge.name=br-custom \
    --opt com.docker.network.bridge.enable_ip_masquerade=true \
    --opt com.docker.network.bridge.enable_icc=true \
    --opt com.docker.network.bridge.host_binding_ipv4=0.0.0.0 \
    custom-bridge

# com.docker.network.bridge.name: custom Linux bridge name
# enable_ip_masquerade: enable IP masquerade for outbound traffic
# enable_icc: allow inter-container communication
# host_binding_ipv4: host IP to bind published ports

# View the bridge kernel interface
ip link show br-custom
ip -6 addr show br-custom
```

## Bridge Network Inspection and Debug

```bash
# Detailed bridge inspection
docker network inspect mybridge

# List all bridge interfaces
ip link show type bridge

# Show bridge forwarding table
bridge fdb show br br-$(docker network inspect mybridge \
    --format "{{.Id}}" | head -c 12)

# Check veth pairs connected to bridge
ip link show | grep "veth" | head -10

# Find which container owns a veth pair
for veth in $(ls /sys/class/net | grep veth); do
    PID=$(docker inspect $(docker ps -q) --format \
        "{{.Id}} {{.Name}}" 2>/dev/null | while read id name; do
        if ls /proc/$(docker inspect --format "{{.State.Pid}}" "$id" 2>/dev/null)/net/if_inet6 2>/dev/null | xargs -I{} grep -l "$veth" 2>/dev/null | grep -q .; then
            echo "$name"
        fi
    done)
    echo "veth: $veth -> container: ${PID:-unknown}"
done
```

## Conclusion

Docker bridge networks support IPv6 through both the default bridge (configured via `fixed-cidr-v6` in `daemon.json`) and user-defined bridges (with explicit `--subnet <ipv6-cidr>` on network create). User-defined bridges are preferred for IPv6 because they support DNS resolution between containers by container name, whereas the default bridge requires manual IP addressing. Set `enable_ip_masquerade=true` for outbound IPv6 internet access from containers. User-defined bridges also allow containers to be connected and disconnected at runtime without restart.
