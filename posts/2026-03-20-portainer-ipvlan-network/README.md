# How to Create an IPvlan Network in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Networking, IPvlan, DevOps

Description: Learn how to create IPvlan networks in Portainer as an alternative to Macvlan that works with stricter network environments.

## Introduction

IPvlan is similar to Macvlan but with a key difference: instead of assigning each container a unique MAC address, IPvlan shares the parent interface's MAC address while giving each container a unique IP. This makes IPvlan work in environments where MAC address filtering or limits per port prevent Macvlan from working (e.g., cloud VMs, some managed switches).

## Prerequisites

- Portainer installed with a connected Docker environment
- Linux kernel 4.2+ (for IPvlan support)
- Understanding of your network topology

## Macvlan vs. IPvlan

| Aspect | Macvlan | IPvlan |
|--------|---------|--------|
| MAC address | Unique per container | Shared (parent's MAC) |
| Promiscuous mode | Required | Not required |
| Cloud VMs | Often blocked | Usually works |
| DHCP per container | Possible | Not possible |
| Modes | bridge, passthru | L2, L3 |

## IPvlan Modes

**L2 mode** (default): Containers appear on the same subnet as the parent interface. Similar behavior to Macvlan.

**L3 mode**: Containers are on isolated subnets, and the host acts as a router. More isolation, more complex routing.

## Step 1: Create IPvlan L2 Network via Portainer

1. Navigate to **Networks** in Portainer.
2. Click **Add network**.
3. Configure:

```
Name:    ipvlan-l2
Driver:  ipvlan
```

4. Network configuration:

```
Subnet:   192.168.1.0/24
Gateway:  192.168.1.1
IP Range: 192.168.1.200/26
```

5. Advanced options:

```
parent: eth0
ipvlan_mode: l2
```

6. Click **Create the network**.

## Step 2: Create IPvlan Network via CLI

```bash
# IPvlan L2 (default):
docker network create \
  --driver ipvlan \
  --subnet 192.168.1.0/24 \
  --gateway 192.168.1.1 \
  --ip-range 192.168.1.200/26 \
  --opt parent=eth0 \
  ipvlan-l2

# IPvlan L3 (advanced routing):
docker network create \
  --driver ipvlan \
  --subnet 10.10.0.0/24 \
  --opt parent=eth0 \
  --opt ipvlan_mode=l3 \
  ipvlan-l3

# Verify:
docker network inspect ipvlan-l2
```

## Step 3: Use IPvlan in Docker Compose

```yaml
# docker-compose.yml with IPvlan L2
version: "3.8"

services:
  # Service with dedicated IP on physical network
  iot-gateway:
    image: myorg/iot-gateway:latest
    restart: unless-stopped
    networks:
      ipvlan-l2:
        ipv4_address: 192.168.1.205   # Fixed IP from the ip-range

  # Web interface accessible on physical network
  grafana:
    image: grafana/grafana:latest
    restart: unless-stopped
    networks:
      ipvlan-l2:
        ipv4_address: 192.168.1.206
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_PASSWORD}

networks:
  ipvlan-l2:
    external: true   # Use the pre-created ipvlan network
```

## Step 4: IPvlan L3 for Advanced Routing

L3 mode is used when you want containers on a different subnet:

```bash
# L3 mode: containers on 10.10.0.0/24 subnet
# Host routes traffic as a router

docker network create \
  --driver ipvlan \
  --subnet 10.10.0.0/24 \
  --opt parent=eth0 \
  --opt ipvlan_mode=l3 \
  ipvlan-l3

# Container gets IP from 10.10.0.0/24:
docker run -d \
  --network ipvlan-l3 \
  --ip 10.10.0.10 \
  myapp:latest
```

For L3 containers to be reachable from the rest of the network, add a static route on the router:

```
# On your network router:
route add 10.10.0.0/24 via <docker-host-ip>
```

## Step 5: IPvlan vs. Macvlan Decision Tree

```
Are you on a cloud VM (AWS EC2, GCP, Azure)?
  → YES: Use IPvlan (Macvlan often blocked by hypervisor)
  → NO: Either works

Does your switch/router allow MAC address learning?
  → YES: Macvlan or IPvlan both work
  → NO (MAC filtering/portfast): Use IPvlan

Do you need DHCP per container?
  → YES: Use Macvlan
  → NO: Either works

Do you need containers on a different subnet?
  → YES: Use IPvlan L3
  → NO: Use Macvlan or IPvlan L2
```

## Step 6: Cloud VM Configuration

On cloud VMs, enable IP forwarding and disable source/destination check:

```bash
# AWS EC2: disable source/destination check via AWS console or CLI
aws ec2 modify-network-interface-attribute \
  --network-interface-id eni-xxxxxxxxxxxx \
  --no-source-dest-check

# Enable IP forwarding on the host:
echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward
# Persistent:
echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

## Step 7: Verify IPvlan Network

```bash
# Test that a container is on the physical network:
docker run -d \
  --name test-ipvlan \
  --network ipvlan-l2 \
  --ip 192.168.1.201 \
  alpine sleep 300

# Container should be reachable from physical network:
ping 192.168.1.201

# Check from inside the container:
docker exec test-ipvlan ip addr show eth0
# Should show: inet 192.168.1.201/24

docker exec test-ipvlan ping 192.168.1.1   # Should reach gateway

# Cleanup:
docker rm -f test-ipvlan
```

## Troubleshooting IPvlan

```bash
# Error: "Error response from daemon: Invalid option: 'parent'"
# → Ensure Docker version supports ipvlan (Docker 1.13+, Linux kernel 4.2+)

# Container not reachable:
# → Check IP range doesn't conflict with DHCP pool
# → Verify container IP is in the configured ip-range
# → Check host firewall (iptables/nftables)

# L3 mode: containers can't reach external network:
# → Add static route on physical router pointing to docker host
# → Enable IP forwarding: sysctl net.ipv4.ip_forward=1
```

## Conclusion

IPvlan networks in Portainer offer an alternative to Macvlan for giving containers direct network presence without unique MAC addresses. They work better in cloud environments, on managed switches with MAC filtering, and in scenarios where the hypervisor blocks promiscuous mode. Choose IPvlan L2 for simple LAN integration and IPvlan L3 for container subnets with the Docker host acting as a router.
