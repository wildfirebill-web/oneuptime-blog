# How to Create a VLAN Interface with ip link add type vlan

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, ip command, iproute2, VLAN, 802.1Q, Networking

Description: Create 802.1Q VLAN subinterfaces on Linux using ip link add type vlan, including VLAN ID assignment, IP configuration, and interface management.

## Introduction

`ip link add type vlan` creates an 802.1Q VLAN subinterface on a physical interface. The VLAN ID (1-4094) tags outgoing frames and filters incoming frames. This is the direct iproute2 approach without using high-level tools like nmcli or Netplan.

## Create a VLAN Interface

```bash
# Create VLAN 10 subinterface on eth0

ip link add link eth0 name eth0.10 type vlan id 10

# Assign an IP address
ip addr add 192.168.10.1/24 dev eth0.10

# Bring it up
ip link set eth0.10 up
```

## Verify the VLAN Interface

```bash
# Show VLAN interface details
ip -d link show eth0.10

# Sample output includes:
# vlan protocol 802.1Q id 10 <REORDER_HDR>

# Check the IP address
ip addr show eth0.10
```

## Create Multiple VLANs on One Interface

```bash
# VLAN 10
ip link add link eth0 name eth0.10 type vlan id 10
ip addr add 192.168.10.1/24 dev eth0.10
ip link set eth0.10 up

# VLAN 20
ip link add link eth0 name eth0.20 type vlan id 20
ip addr add 192.168.20.1/24 dev eth0.20
ip link set eth0.20 up

# VLAN 30
ip link add link eth0 name eth0.30 type vlan id 30
ip addr add 192.168.30.1/24 dev eth0.30
ip link set eth0.30 up
```

## Custom Interface Name

```bash
# VLAN interface does not have to be named parent.ID
ip link add link eth0 name mgmt type vlan id 100
ip addr add 10.100.0.1/24 dev mgmt
ip link set mgmt up
```

## VLAN with 802.1ad (QinQ)

```bash
# Create outer VLAN (S-VLAN) with 802.1ad
ip link add link eth0 name eth0.100 type vlan id 100 proto 802.1ad

# Create inner VLAN (C-VLAN) on top
ip link add link eth0.100 name eth0.100.200 type vlan id 200 proto 802.1Q
```

## Load the 8021q Module

```bash
# Ensure the kernel module is loaded
modprobe 8021q

# Verify
lsmod | grep 8021q
```

## Delete a VLAN Interface

```bash
ip link set eth0.10 down
ip link delete eth0.10
```

## Conclusion

`ip link add link <parent> name <name> type vlan id <vid>` creates a VLAN subinterface. The interface handles 802.1Q tagging automatically. Load the `8021q` kernel module first. These changes are not persistent - use Netplan, nmcli, or systemd-networkd for permanent VLAN configuration.
