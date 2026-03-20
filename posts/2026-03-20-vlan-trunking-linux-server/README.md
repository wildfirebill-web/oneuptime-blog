# How to Configure VLAN Trunking on a Linux Server

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, VLAN, Trunking, 802.1Q, Networking, Network Configuration, iproute2

Description: Configure a Linux server as a VLAN trunk host by creating multiple tagged VLAN subinterfaces on a physical interface connected to a switch trunk port.

## Introduction

VLAN trunking allows a single physical link to carry traffic for multiple VLANs simultaneously using 802.1Q tagging. On the Linux side, this means creating subinterfaces for each VLAN. On the switch side, the connected port must be configured as a trunk port allowing the required VLANs. This guide covers the Linux configuration.

## Prerequisites

- Physical interface connected to a switch trunk port
- Switch configured with trunk port allowing VLANs 10, 20, 30 (example)
- `8021q` module loaded
- Root access

## Linux Trunk Configuration

The physical interface acts as the "trunk" - it does not need an IP address. Each VLAN subinterface handles traffic for its VLAN:

```bash
# Load 802.1Q module

modprobe 8021q

# Bring up parent interface without an IP
ip link set eth0 up

# Create VLAN subinterfaces for each VLAN on the trunk
ip link add link eth0 name eth0.10 type vlan id 10
ip link add link eth0 name eth0.20 type vlan id 20
ip link add link eth0 name eth0.30 type vlan id 30

# Assign IPs to each VLAN
ip addr add 10.10.0.1/24 dev eth0.10
ip addr add 10.20.0.1/24 dev eth0.20
ip addr add 10.30.0.1/24 dev eth0.30

# Bring up all VLAN interfaces
ip link set eth0.10 up
ip link set eth0.20 up
ip link set eth0.30 up
```

## Switch Port Configuration Reference

For a Cisco switch (for reference only - switch config is not done on Linux):

```text
! Configure the connected switch port as a trunk
interface GigabitEthernet0/1
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30
 switchport trunk native vlan 1
```

## Native VLAN Handling

The native VLAN carries untagged traffic on a trunk port. On Linux, the parent interface handles untagged (native) traffic:

```bash
# Assign an IP to the parent interface for native VLAN traffic
# Only if you want the Linux host to participate in the native VLAN
ip addr add 10.1.0.1/24 dev eth0
```

## Persistent Configuration with Netplan

```yaml
# /etc/netplan/01-trunk.yaml
network:
  version: 2
  renderer: networkd

  ethernets:
    eth0:
      dhcp4: false   # No IP on trunk interface

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
```

## Verify Trunk is Working

```bash
# Check all VLAN interfaces are up
ip link show type vlan

# Verify traffic on the trunk (should see tagged frames)
tcpdump -i eth0 -e vlan -n -c 20

# Check routing table
ip route show
```

## Enable Inter-VLAN Routing

```bash
# Enable IP forwarding to route between VLANs
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.d/99-routing.conf
sysctl -p /etc/sysctl.d/99-routing.conf
```

## Conclusion

VLAN trunking on Linux is implemented by leaving the physical interface without an IP and creating tagged VLAN subinterfaces for each VLAN ID. The physical interface and switch port must both be configured for trunk mode. This allows a single cable to carry dozens of VLANs between the Linux host and the switch.
