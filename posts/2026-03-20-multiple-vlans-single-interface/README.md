# How to Configure Multiple VLANs on a Single Physical Interface

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, VLAN, 802.1Q, Trunking, iproute2, Networking, Network Segmentation

Description: Configure multiple VLAN subinterfaces on a single physical network interface to segment traffic across several VLANs using a single trunk port.

## Introduction

A single physical interface connected to a switch trunk port can carry traffic for multiple VLANs simultaneously. Each VLAN is represented as a separate subinterface (e.g., `eth0.10`, `eth0.20`), each with its own IP address, routing, and firewall rules. This is the standard configuration for Linux routers and multi-tenant servers.

## Prerequisites

- Physical interface connected to a switch trunk port with the VLANs allowed
- `8021q` kernel module loaded
- Root access

## Load the Module

```bash
modprobe 8021q
echo "8021q" > /etc/modules-load.d/8021q.conf
```

## Create Multiple VLANs Using iproute2

```bash
# Bring up the parent interface (no IP needed on the parent)
ip link set eth0 up

# VLAN 10 - Management
ip link add link eth0 name eth0.10 type vlan id 10
ip addr add 10.10.0.1/24 dev eth0.10
ip link set eth0.10 up

# VLAN 20 - Production servers
ip link add link eth0 name eth0.20 type vlan id 20
ip addr add 10.20.0.1/24 dev eth0.20
ip link set eth0.20 up

# VLAN 30 - DMZ
ip link add link eth0 name eth0.30 type vlan id 30
ip addr add 10.30.0.1/24 dev eth0.30
ip link set eth0.30 up

# VLAN 40 - IoT devices
ip link add link eth0 name eth0.40 type vlan id 40
ip addr add 10.40.0.1/24 dev eth0.40
ip link set eth0.40 up
```

## Verify All VLANs

```bash
# List all VLAN interfaces
ip link show type vlan

# Show IPs on all interfaces
ip addr show | grep -A 3 eth0\.

# Check routing table (one route per VLAN subnet)
ip route show
```

## Persistent Configuration with Netplan

```yaml
# /etc/netplan/01-vlans.yaml
network:
  version: 2
  renderer: networkd

  ethernets:
    eth0:
      dhcp4: false

  vlans:
    eth0.10:
      id: 10
      link: eth0
      addresses: [10.10.0.1/24]

    eth0.20:
      id: 20
      link: eth0
      addresses: [10.20.0.1/24]

    eth0.30:
      id: 30
      link: eth0
      addresses: [10.30.0.1/24]

    eth0.40:
      id: 40
      link: eth0
      dhcp4: true
```

## Enable Inter-VLAN Routing

```bash
# Enable IP forwarding to route between VLANs
sysctl -w net.ipv4.ip_forward=1

# Each VLAN subnet is directly connected — routes are automatic
# Hosts on 10.10.0.0/24 can reach 10.20.0.0/24 via the Linux router
```

## Apply Per-VLAN Firewall Rules

```bash
# Block VLAN 40 (IoT) from accessing VLAN 20 (production)
iptables -A FORWARD -i eth0.40 -o eth0.20 -j DROP
iptables -A FORWARD -i eth0.20 -o eth0.40 -m state --state ESTABLISHED,RELATED -j ACCEPT
```

## Conclusion

Multiple VLANs on a single physical interface requires the switch to be configured in trunk mode for all the VLANs you want to carry. On the Linux side, create a subinterface per VLAN with a unique IP prefix. Enable IP forwarding to route between VLANs and add firewall rules to control inter-VLAN access. This configuration is the foundation of a Linux-based multi-VLAN router.
