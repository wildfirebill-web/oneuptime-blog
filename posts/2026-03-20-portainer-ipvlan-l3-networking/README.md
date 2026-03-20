# How to Configure IPvlan L3 Mode for Container Routing in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, IPvlan, L3 Mode, Docker Networking, Container Routing, Advanced Networking

Description: Learn how to configure Docker IPvlan L3 mode in Portainer for layer-3 routed container networking without ARP broadcast traffic.

---

IPvlan L3 mode operates at the network layer. Containers get IP addresses in a separate subnet, and traffic is routed between containers and the external network via the host's routing table. This eliminates ARP broadcasts and is suitable for large-scale deployments or when containers need custom subnets.

## IPvlan L3 vs L2

| Feature | IPvlan L2 | IPvlan L3 |
|---------|-----------|-----------|
| Layer | Data Link (Layer 2) | Network (Layer 3) |
| ARP | Yes, between containers | No ARP (host routes traffic) |
| Gateway | Uses LAN router | Host acts as router |
| Container subnet | Must be in LAN subnet | Can be any subnet |
| External routing | Via LAN router | Requires static routes on upstream router |

## Creating an IPvlan L3 Network

L3 containers live in their own subnet — no gateway is specified because the host routes traffic:

```bash
docker network create \
  --driver ipvlan \
  --opt ipvlan_mode=l3 \
  --subnet 172.16.100.0/24 \
  --opt parent=eth0 \
  ipvlan_l3_net
```

Note: No `--gateway` flag — in L3 mode, the host kernel handles routing.

## Using IPvlan L3 in a Stack

```yaml
version: "3.8"

services:
  api:
    image: my-api:latest
    networks:
      ipvlan_l3_net:
        ipv4_address: 172.16.100.10

  worker:
    image: my-worker:latest
    networks:
      ipvlan_l3_net:
        ipv4_address: 172.16.100.11

networks:
  ipvlan_l3_net:
    external: true
```

## Enabling External Access

Containers in L3 subnets are isolated by default. For external access, add a route on the upstream router pointing to the Docker host:

```bash
# On your router/gateway (example for Linux router):
sudo ip route add 172.16.100.0/24 via 192.168.1.100   # 192.168.1.100 = Docker host IP

# On the Docker host, enable IP forwarding
sudo sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.conf
```

For the host itself to reach L3 containers, add a direct route:

```bash
sudo ip route add 172.16.100.0/24 dev eth0
```

## Multiple L3 Subnets

Create multiple isolated L3 networks on the same interface for tenant isolation:

```bash
# Tenant A network
docker network create \
  --driver ipvlan --opt ipvlan_mode=l3 \
  --subnet 172.16.101.0/24 --opt parent=eth0 \
  tenant_a_net

# Tenant B network
docker network create \
  --driver ipvlan --opt ipvlan_mode=l3 \
  --subnet 172.16.102.0/24 --opt parent=eth0 \
  tenant_b_net
```

Containers on different subnets cannot communicate by default, providing natural tenant isolation without iptables rules.

## Verifying L3 Routing

```bash
# Check the container's routing table
docker exec -it $(docker ps -qf name=api) ip route

# Expected output:
# default via 0.0.0.0 dev eth0   (no gateway — host routes)
# 172.16.100.0/24 dev eth0 proto kernel scope link

# Verify host sees the route
ip route show | grep 172.16.100
```

## When to Use L3

IPvlan L3 is suitable when:

- You need containers in custom subnets that don't overlap with the LAN.
- You have many containers and want to eliminate ARP broadcast storm issues.
- You control the upstream router and can add static routes.
- You need isolated multi-tenant networking.

For most home lab and small production setups, IPvlan L2 or macvlan is simpler and doesn't require upstream router configuration.
