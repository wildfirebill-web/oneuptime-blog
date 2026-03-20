# How to Create a Network Bridge on Linux Using iproute2

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, Network Bridge, iproute2, Networking, Virtualization, L2 Switching

Description: Create a Linux network bridge using ip link commands to connect physical interfaces and virtual machine tap interfaces at Layer 2.

## Introduction

A Linux network bridge operates as a software Layer 2 switch, forwarding frames between connected interfaces based on MAC addresses. Bridges are used for KVM virtualization, container networking, and connecting network segments. The `ip link` command from iproute2 provides a simple way to create and manage bridges.

## Create a Bridge Interface

```bash
# Create a new bridge named br0
ip link add br0 type bridge

# Bring it up
ip link set br0 up
```

## Add Physical Interface to the Bridge

When you add a physical interface to a bridge, it becomes a "bridge port" — it no longer handles IP traffic directly:

```bash
# Remove the IP from eth0 first (if it has one)
ip addr flush dev eth0

# Add eth0 to the bridge
ip link set eth0 master br0
ip link set eth0 up

# Assign the IP to the bridge instead
ip addr add 192.168.1.100/24 dev br0
ip route add default via 192.168.1.1
```

## Verify the Bridge

```bash
# Show bridge interfaces
bridge link show

# Show bridge details
ip -d link show br0

# Show bridge forwarding database
bridge fdb show br br0
```

## Add Multiple Interfaces

```bash
# Add a second interface (for bridging two physical networks)
ip link set eth1 master br0
ip link set eth1 up

# List all ports on the bridge
bridge link show
```

## Full Bridge Setup Script

```bash
#!/bin/bash
# create-bridge.sh

# Create bridge
ip link add br0 type bridge
ip link set br0 up

# Flush IP from physical interface
ip addr flush dev eth0

# Add physical interface to bridge
ip link set eth0 master br0
ip link set eth0 up

# Assign IP to bridge
ip addr add 192.168.1.100/24 dev br0
ip route add default via 192.168.1.1

echo "Bridge created:"
bridge link show
ip addr show br0
```

## Disable STP for Better Performance (Optional)

Spanning Tree Protocol (STP) adds delay when interfaces come up. Disable it if you don't have loops:

```bash
ip link set br0 type bridge stp_state 0
```

## Delete the Bridge

```bash
# Remove interface from bridge first
ip link set eth0 nomaster

# Delete the bridge
ip link delete br0
```

## Persistent Configuration (Netplan)

```yaml
network:
  version: 2
  ethernets:
    eth0: {dhcp4: false}
  bridges:
    br0:
      interfaces: [eth0]
      addresses: [192.168.1.100/24]
      parameters:
        stp: false
```

## Conclusion

Creating a Linux bridge with iproute2 requires `ip link add br0 type bridge`, adding interfaces with `ip link set <dev> master br0`, and assigning the IP to the bridge (not the physical interface). Bridges are transparent at Layer 2 and forward frames based on learned MAC addresses in the forwarding database.
