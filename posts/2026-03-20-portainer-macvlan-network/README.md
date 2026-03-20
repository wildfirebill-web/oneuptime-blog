# How to Create a Macvlan Network in Portainer - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Networking, Macvlan, DevOps

Description: Learn how to create a Macvlan network in Portainer to give containers their own MAC addresses and direct access to the physical network.

## Introduction

Macvlan networks give each container a unique MAC address and an IP address on the physical network, making them appear as standalone physical machines to the network. This is useful for containers that need to be directly reachable on the network (legacy applications, network monitoring tools, IoT gateways) without port forwarding.

## Prerequisites

- Portainer installed with a connected Docker environment
- A network interface that supports promiscuous mode
- Network subnet and gateway information

## When to Use Macvlan

| Use Case | Why Macvlan |
|----------|------------|
| Container needs a dedicated IP on the LAN | Assigned from physical network pool |
| Legacy app that hardcodes its IP | Give it a specific IP |
| Network monitoring (packet capture) | Direct network access |
| IoT gateway (OPC-UA, Modbus TCP) | PLC can reach container directly |
| DNS server needing port 53 | No port mapping conflict |

## Step 1: Enable Promiscuous Mode on the Host Interface

Macvlan requires the host interface to be in promiscuous mode:

```bash
# Enable promiscuous mode on eth0:

sudo ip link set eth0 promisc on

# Verify:
ip link show eth0 | grep promisc
# Should show: PROMISC

# Make it persistent (add to /etc/rc.local or systemd):
sudo tee /etc/systemd/system/macvlan-promisc.service << 'EOF'
[Unit]
Description=Enable promiscuous mode for Macvlan
After=network.target

[Service]
Type=oneshot
ExecStart=/sbin/ip link set eth0 promisc on
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl enable macvlan-promisc
sudo systemctl start macvlan-promisc
```

## Step 2: Create Macvlan Network via Portainer

1. Navigate to **Networks** in Portainer.
2. Click **Add network**.
3. Configure:

```text
Name:       macvlan-lan
Driver:     macvlan
```

4. Under **Network configuration**:

```bash
Subnet:       192.168.1.0/24   (your physical network subnet)
Gateway:      192.168.1.1      (your router/gateway)
IP Range:     192.168.1.128/26 (range for Docker containers: .128-.191)
```

5. Under **Advanced configuration** > **Options**:

```text
parent: eth0   (the physical network interface)
```

6. Click **Create the network**.

## Step 3: Create Macvlan Network via CLI

```bash
# Create a macvlan network:
docker network create \
  --driver macvlan \
  --subnet 192.168.1.0/24 \
  --gateway 192.168.1.1 \
  --ip-range 192.168.1.128/26 \
  --opt parent=eth0 \
  macvlan-lan

# Verify:
docker network ls | grep macvlan
docker network inspect macvlan-lan
```

## Step 4: Assign IPs to Containers

```yaml
# docker-compose.yml with macvlan
version: "3.8"

services:
  # Container gets IP 192.168.1.130 from macvlan network
  dns-server:
    image: coredns/coredns:latest
    restart: unless-stopped
    networks:
      macvlan-lan:
        ipv4_address: 192.168.1.130   # Fixed IP on physical network
    volumes:
      - ./Corefile:/Corefile:ro

  # OPC-UA server accessible directly by PLCs
  opcua-server:
    image: myorg/opcua-server:latest
    restart: unless-stopped
    networks:
      macvlan-lan:
        ipv4_address: 192.168.1.131   # PLCs connect to this IP directly

  # Network monitoring tool
  ntopng:
    image: ntop/ntopng:stable
    restart: unless-stopped
    networks:
      macvlan-lan:
        ipv4_address: 192.168.1.132
    cap_add:
      - NET_ADMIN
      - NET_RAW

networks:
  macvlan-lan:
    external: true   # Use the pre-created macvlan network
```

## Step 5: Host-to-Container Communication with Macvlan

A limitation of macvlan: the host cannot directly communicate with containers on the macvlan network (the host interface doesn't route to child macvlan interfaces).

Workaround: create a macvlan sub-interface on the host:

```bash
# Create macvlan interface on host for host-to-container communication:
sudo ip link add macvlan-host link eth0 type macvlan mode bridge
sudo ip addr add 192.168.1.200/24 dev macvlan-host
sudo ip link set macvlan-host up

# Add route for the container IP range:
sudo ip route add 192.168.1.128/26 dev macvlan-host

# Now the host can reach containers:
ping 192.168.1.130   # DNS server container
```

## Step 6: 802.1q VLAN-Based Macvlan

For VLAN-tagged macvlan (one physical interface, multiple VLANs):

```bash
# Create VLAN interface first:
sudo ip link add link eth0 name eth0.100 type vlan id 100
sudo ip link set eth0.100 up

# Create macvlan on VLAN interface:
docker network create \
  --driver macvlan \
  --subnet 10.100.0.0/24 \
  --gateway 10.100.0.1 \
  --opt parent=eth0.100 \
  macvlan-vlan100
```

## Step 7: Assign Static IPs for Critical Services

```bash
# Container with fixed IP:
docker run -d \
  --name critical-service \
  --network macvlan-lan \
  --ip 192.168.1.130 \
  myorg/critical-service:latest

# Verify the IP assignment:
docker exec critical-service ip addr show eth0
# inet 192.168.1.130/24 brd 192.168.1.255 scope global eth0

# Test from another machine on the network:
ping 192.168.1.130
curl http://192.168.1.130:8080/health
```

## Step 8: Macvlan vs. Bridge: When to Choose

```text
Use Bridge when:
  - Containers on the same host need to talk to each other
  - External access via port mapping is sufficient
  - Standard web applications

Use Macvlan when:
  - Container needs its own IP on the physical network
  - Other devices (PLCs, IoT, legacy apps) need to connect directly
  - Running services that conflict with host ports (port 53, 443)
  - Network monitoring requiring raw packet access
```

## Troubleshooting

```bash
# Container not getting network connectivity:
# Check promiscuous mode is enabled:
ip link show eth0 | grep PROMISC

# Container IP not reachable from network:
# Check the IP is in the correct range and not in use:
arp-scan 192.168.1.128-192.168.1.191

# Container can't reach gateway:
# On VMs (VMware/VirtualBox), enable promiscuous mode in VM settings
# Check hypervisor networking settings allow MAC spoofing
```

## Conclusion

Macvlan networks in Portainer give containers a first-class presence on your physical network with their own MAC and IP addresses. This is the right choice for containers that need to be reachable directly from the network without port mapping - particularly useful for DNS servers, industrial OPC-UA servers, and network monitoring tools. Remember that the host needs promiscuous mode enabled on the parent interface, and host-to-container communication requires a macvlan sub-interface workaround.
