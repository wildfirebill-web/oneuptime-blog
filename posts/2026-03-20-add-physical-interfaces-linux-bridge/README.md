# How to Add Physical Interfaces to a Linux Bridge

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, Network Bridge, iproute2, Networking, L2 Switching, Bridging

Description: Add one or more physical network interfaces to a Linux bridge to create a software switch that forwards frames at Layer 2 between connected networks.

## Introduction

Adding physical interfaces to a bridge makes them "bridge ports." The bridge learns MAC addresses and forwards frames between ports, creating a virtual switch. This is used for connecting virtual machines to physical networks, bridging two network segments, and aggregating interfaces.

## Add a Physical Interface to an Existing Bridge

```bash
# First flush the IP from the physical interface (if it has one)
ip addr flush dev eth0

# Ensure the interface is up
ip link set eth0 up

# Add eth0 as a port of br0
ip link set eth0 master br0

# Verify the port was added
bridge link show
```

## Check Port State

When STP is disabled, ports immediately enter forwarding state. With STP enabled, ports go through listening → learning → forwarding:

```bash
# Show all bridge ports and their states
bridge link show

# Example output:
# 2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 master br0 state forwarding priority 32 cost 4
```

## Add Multiple Ports

```bash
# Add eth0 and eth1 to the bridge
ip link set eth0 master br0
ip link set eth1 master br0

# Verify both ports are on the bridge
bridge link show br br0
```

## Remove a Port from the Bridge

```bash
# Remove eth0 from the bridge
ip link set eth0 nomaster

# Re-assign IP to eth0 if needed
ip addr add 192.168.1.100/24 dev eth0
```

## Set Port Priority

Bridge port priority influences which port STP selects as the designated port:

```bash
# Set port priority (lower = more preferred, range 0-63, default 32)
bridge link set dev eth0 priority 16
```

## Set Port Cost

Port cost influences STP path selection (lower cost = preferred):

```bash
# Set path cost for eth0 (lower = preferred)
bridge link set dev eth0 cost 4
```

## Configure Port as Access Port (for VLAN-aware bridges)

```bash
# Enable VLAN filtering on the bridge first
ip link set br0 type bridge vlan_filtering 1

# Add eth0 as an access port for VLAN 10
bridge vlan add dev eth0 vid 10 pvid untagged

# Remove the default VLAN 1
bridge vlan del dev eth0 vid 1
```

## Verify Bridge Forwarding Database

After adding ports and generating traffic, the bridge learns MAC addresses:

```bash
# Show learned MAC addresses
bridge fdb show br br0

# Show only dynamic entries (learned from traffic)
bridge fdb show br br0 | grep "master br0"
```

## Conclusion

Adding physical interfaces to a Linux bridge with `ip link set <dev> master <bridge>` creates bridge ports that forward traffic at Layer 2. Always flush the IP from physical interfaces before adding them to a bridge — the IP moves to the bridge interface itself. Multiple ports create a virtual switch where the bridge forwards frames based on learned MAC addresses.
