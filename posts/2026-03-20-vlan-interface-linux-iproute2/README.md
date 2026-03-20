# How to Create a VLAN Interface on Linux Using iproute2

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, VLAN, iproute2, 802.1Q, Networking, Network Configuration

Description: Create 802.1Q VLAN subinterfaces on Linux using the ip link command from iproute2 to segment network traffic on a physical interface.

## Introduction

VLANs (Virtual LANs) allow you to segment network traffic on a single physical interface using 802.1Q tagging. Linux supports VLAN interfaces natively with the `8021q` kernel module. Each VLAN is represented as a subinterface (e.g., `eth0.100`) that sends and receives tagged frames for a specific VLAN ID.

## Prerequisites

- Linux with iproute2 installed
- The `8021q` kernel module loaded
- A physical interface connected to a VLAN-aware switch
- Root access

## Step 1: Load the 8021q Kernel Module

```bash
# Load the VLAN kernel module
modprobe 8021q

# Verify it is loaded
lsmod | grep 8021q

# Make it load at boot
echo "8021q" >> /etc/modules-load.d/vlan.conf
```

## Step 2: Create a VLAN Interface

```bash
# Create VLAN 100 on physical interface eth0
# The new interface will be named eth0.100
ip link add link eth0 name eth0.100 type vlan id 100

# Verify the VLAN interface was created
ip link show eth0.100
```

## Step 3: Assign an IP Address

```bash
# Assign an IPv4 address to the VLAN interface
ip addr add 192.168.100.1/24 dev eth0.100

# Bring the VLAN interface up
ip link set eth0.100 up

# Also ensure the parent interface is up
ip link set eth0 up
```

## Step 4: Verify the Configuration

```bash
# Show the VLAN interface with its IP
ip addr show eth0.100

# Show the VLAN info (protocol, ID, parent)
ip -d link show eth0.100
```

## Create Multiple VLANs on One Interface

```bash
# VLAN 10 for management
ip link add link eth0 name eth0.10 type vlan id 10
ip addr add 10.10.0.1/24 dev eth0.10
ip link set eth0.10 up

# VLAN 20 for production
ip link add link eth0 name eth0.20 type vlan id 20
ip addr add 10.20.0.1/24 dev eth0.20
ip link set eth0.20 up

# VLAN 30 for DMZ
ip link add link eth0 name eth0.30 type vlan id 30
ip addr add 10.30.0.1/24 dev eth0.30
ip link set eth0.30 up
```

## Use a Custom Interface Name

You do not have to use the `parent.vlan` naming convention:

```bash
# Create VLAN 200 with a custom name
ip link add link eth0 name mgmt type vlan id 200
ip addr add 172.16.0.1/24 dev mgmt
ip link set mgmt up
```

## Delete a VLAN Interface

```bash
# Remove the VLAN interface
ip link delete eth0.100
```

## Make VLAN Persistent

Rules added with `ip link` are not persistent. Use `/etc/network/interfaces`, Netplan, or systemd-networkd to persist VLANs (covered in separate guides). For a quick persistence approach:

```bash
# Save setup commands in a script and add to systemd or rc.local
cat >> /etc/rc.local << 'EOF'
modprobe 8021q
ip link add link eth0 name eth0.100 type vlan id 100
ip addr add 192.168.100.1/24 dev eth0.100
ip link set eth0.100 up
ip link set eth0 up
EOF
```

## Conclusion

Creating a VLAN interface with iproute2 requires loading the `8021q` module, running `ip link add ... type vlan id <ID>`, assigning an IP, and bringing the interface up. VLANs allow multiple logical networks over a single physical interface and are fundamental to modern network segmentation. For production systems, persist the configuration using Netplan, nmcli, or systemd-networkd.
