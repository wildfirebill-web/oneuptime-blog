# How to Configure VLANs on Top of a Bond Interface

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, VLAN, Network Bonding, 802.1Q, Networking, High Availability, Link Aggregation

Description: Stack 802.1Q VLAN subinterfaces on top of a Linux bond interface to combine link redundancy with VLAN network segmentation.

## Introduction

Combining bonding and VLANs is one of the most common production Linux networking patterns. The bond provides link redundancy or aggregation, and VLAN subinterfaces on the bond provide network segmentation. VLAN configuration on a bond is identical to VLAN configuration on a physical interface.

## Prerequisites

- Bond interface configured and active (e.g., `bond0`)
- `8021q` kernel module loaded
- Root access

## Add VLANs to bond0 (iproute2)

```bash
# Load 8021q module

modprobe 8021q

# Ensure bond0 is up (with no IP for trunk mode)
ip link set bond0 up

# Create VLAN 10 on the bond
ip link add link bond0 name bond0.10 type vlan id 10
ip addr add 10.10.0.1/24 dev bond0.10
ip link set bond0.10 up

# Create VLAN 20 on the bond
ip link add link bond0 name bond0.20 type vlan id 20
ip addr add 10.20.0.1/24 dev bond0.20
ip link set bond0.20 up

# Create VLAN 30 on the bond
ip link add link bond0 name bond0.30 type vlan id 30
ip addr add 10.30.0.1/24 dev bond0.30
ip link set bond0.30 up
```

## Persistent Configuration: Netplan

```yaml
# /etc/netplan/01-bond-vlans.yaml
network:
  version: 2
  renderer: networkd

  ethernets:
    eth0: {dhcp4: false}
    eth1: {dhcp4: false}

  bonds:
    bond0:
      interfaces: [eth0, eth1]
      parameters:
        mode: active-backup
        mii-monitor-interval: 100
      dhcp4: false   # No IP on bond0 - it's a trunk

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

## Persistent Configuration: nmcli (RHEL)

```bash
# Create the bond
nmcli connection add type bond con-name bond0 ifname bond0 \
    bond.options "mode=active-backup,miimon=100"

nmcli connection add type ethernet con-name bond0-eth0 ifname eth0 master bond0
nmcli connection add type ethernet con-name bond0-eth1 ifname eth1 master bond0

# Add VLAN on top of bond
nmcli connection add type vlan con-name bond0.10 dev bond0 id 10 \
    ipv4.addresses 10.10.0.1/24 ipv4.method manual

nmcli connection add type vlan con-name bond0.20 dev bond0 id 20 \
    ipv4.addresses 10.20.0.1/24 ipv4.method manual

nmcli connection up bond0
nmcli connection up bond0.10
nmcli connection up bond0.20
```

## Verify the Stack

```bash
# Check bond status
cat /proc/net/bonding/bond0 | grep "Currently Active Slave"

# Check VLAN interfaces
ip link show type vlan

# Verify IPs
ip addr show bond0.10
ip addr show bond0.20

# Test VLAN connectivity
ping -I bond0.10 10.10.0.254
```

## Failover Test

```bash
# Bring down the active slave
ip link set eth0 down

# Bond fails over to eth1 - all VLANs remain operational
ping -I bond0.10 10.10.0.254 -c 5
ping -I bond0.20 10.20.0.254 -c 5
```

## Conclusion

VLANs on bond interfaces are configured identically to VLANs on physical interfaces - the bond acts as the parent device. VLAN subinterfaces inherit the bond's failover behavior: when the bond switches slaves, all VLAN traffic continues uninterrupted. Netplan and nmcli both support this configuration natively with their respective `vlans` and `bonds` sections.
