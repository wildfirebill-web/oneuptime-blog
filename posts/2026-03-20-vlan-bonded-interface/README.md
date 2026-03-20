# How to Add a VLAN to a Bonded Interface

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, VLAN, Bonding, 802.1Q, Network Bonding, Networking, High Availability

Description: Create VLAN subinterfaces on top of a Linux bonded interface to combine link aggregation with VLAN segmentation for high-availability tagged networking.

## Introduction

Combining bonding with VLANs is a common production pattern: bonding provides link redundancy or aggregation, while VLANs segment traffic. You create the bond first, then add VLAN subinterfaces on top of it — exactly as you would with a physical interface.

## Prerequisites

- A configured bond interface (e.g., `bond0`)
- `8021q` kernel module loaded
- Root access

## Quick Bond Setup (if not already configured)

```bash
# Load bonding and 8021q modules
modprobe bonding
modprobe 8021q

# Create bond0 in active-backup mode
ip link add bond0 type bond mode active-backup

# Add slave interfaces
ip link set eth0 master bond0
ip link set eth1 master bond0

# Bring up the bond (no IP needed on bond0 itself for VLAN trunking)
ip link set bond0 up
```

## Add VLAN Subinterfaces to the Bond

```bash
# Create VLAN 10 on bond0
ip link add link bond0 name bond0.10 type vlan id 10
ip addr add 10.10.0.1/24 dev bond0.10
ip link set bond0.10 up

# Create VLAN 20 on bond0
ip link add link bond0 name bond0.20 type vlan id 20
ip addr add 10.20.0.1/24 dev bond0.20
ip link set bond0.20 up

# Create VLAN 30 on bond0
ip link add link bond0 name bond0.30 type vlan id 30
ip addr add 10.30.0.1/24 dev bond0.30
ip link set bond0.30 up
```

## Persistent Configuration with Netplan

```yaml
# /etc/netplan/01-bond-vlan.yaml
network:
  version: 2
  renderer: networkd

  ethernets:
    eth0:
      dhcp4: false
    eth1:
      dhcp4: false

  bonds:
    bond0:
      interfaces: [eth0, eth1]
      parameters:
        mode: active-backup
        mii-monitor-interval: 100
      dhcp4: false

  vlans:
    bond0.10:
      id: 10
      link: bond0
      addresses: [10.10.0.1/24]

    bond0.20:
      id: 20
      link: bond0
      addresses: [10.20.0.1/24]

    bond0.30:
      id: 30
      link: bond0
      addresses: [10.30.0.1/24]
```

## Verify the Configuration

```bash
# Check bond status
cat /proc/net/bonding/bond0

# Check VLAN interfaces
ip link show type vlan

# Verify IPs
ip addr show bond0.10
ip addr show bond0.20

# Test connectivity
ping -I bond0.10 10.10.0.254
```

## Failover Behavior

When the active bond interface fails:
1. The bond switches to the backup interface automatically
2. All VLAN subinterfaces remain up (they depend on bond0, not individual slaves)
3. Traffic on all VLANs continues without reconfiguration

```bash
# Simulate a failover by taking down the active interface
ip link set eth0 down
cat /proc/net/bonding/bond0  # eth1 should now be active
```

## Conclusion

VLANs on bonded interfaces combine the redundancy of link bonding with the segmentation of 802.1Q VLANs. Create the bond first, then add VLAN subinterfaces to the bond interface. Failover is transparent to the VLAN configuration — all VLAN interfaces remain operational as long as at least one bond slave is active.
