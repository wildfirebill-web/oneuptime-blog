# How to Configure a VM Host with Bonds and VLANs Using Netplan

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Netplan, VLAN, Bonding, KVM, Linux, Networking, Virtualization

Description: Build a production VM host network with bonded NICs and VLAN-tagged bridges using Netplan to provide isolated networks for virtual machines.

## Introduction

A VM host typically needs high-availability bonded uplinks and VLAN-separated networks for different VM tenants or functions (management, storage, public traffic). Netplan can express this entire topology in a single YAML file.

## Target Topology

```
Physical NICs: eth0 + eth1
      ↓ bond (LACP)
    bond0
      ↓ VLAN tagging
  bond0.100 (Management VLAN) → bridge br-mgmt
  bond0.200 (VM traffic VLAN) → bridge br-vm
```

## Full Netplan Configuration

```yaml
# /etc/netplan/01-vm-host.yaml
network:
  version: 2
  renderer: networkd

  ethernets:
    eth0:
      dhcp4: false    # Managed by bond
    eth1:
      dhcp4: false    # Managed by bond

  bonds:
    bond0:
      interfaces: [eth0, eth1]
      parameters:
        mode: 802.3ad           # LACP for active-active aggregation
        lacp-rate: fast
        mii-monitor-interval: 100
      dhcp4: false
      mtu: 9000                 # Jumbo frames for VM traffic

  vlans:
    bond0.100:
      id: 100                   # Management VLAN tag
      link: bond0
      dhcp4: false
    bond0.200:
      id: 200                   # VM traffic VLAN tag
      link: bond0
      dhcp4: false

  bridges:
    br-mgmt:
      interfaces: [bond0.100]   # Bridge on management VLAN
      addresses:
        - 10.100.0.1/24
      routes:
        - to: default
          via: 10.100.0.254
      nameservers:
        addresses: [10.100.0.10]
      parameters:
        stp: false              # Disable STP on a simple host bridge
        forward-delay: 0

    br-vm:
      interfaces: [bond0.200]   # Bridge on VM traffic VLAN
      dhcp4: false
      parameters:
        stp: false
        forward-delay: 0
```

## Applying the Configuration

```bash
# Test before committing (reverts automatically if not confirmed)
sudo netplan try

# Apply permanently
sudo netplan apply
```

## Verifying the Setup

```bash
# Check bond status
cat /proc/net/bonding/bond0

# Verify VLAN interfaces
ip link show type vlan

# Verify bridges
bridge link show
```

## Attaching VMs to the Bridge

When creating VMs with libvirt or QEMU, attach them to `br-vm` for network connectivity:

```bash
# Example: attach a QEMU VM NIC to br-vm
-netdev bridge,id=net0,br=br-vm
```

## Conclusion

Netplan's YAML-based configuration makes it straightforward to express complex VM host network topologies. Using bonds ensures NIC failover, VLANs provide isolation, and bridges give VMs access to the physical network.
