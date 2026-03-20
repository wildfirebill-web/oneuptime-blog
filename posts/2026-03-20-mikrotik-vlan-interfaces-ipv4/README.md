# How to Configure VLAN Interfaces for IPv4 on MikroTik

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MikroTik, RouterOS, VLAN, IPv4, Trunking, Switching, 802.1Q

Description: Configure 802.1Q VLAN interfaces on MikroTik RouterOS, assign IPv4 addresses, set up a trunk port, and configure bridge-based switching for a router-on-a-stick design.

## Introduction

MikroTik supports VLANs on physical interfaces and bridges. The modern recommended approach (RouterOS v7) uses bridge VLAN filtering. The older approach creates VLAN sub-interfaces directly on the physical port for router-on-a-stick inter-VLAN routing.

## Router-on-a-Stick (VLAN Sub-Interfaces)

```mikrotik
# Trunk port receives tagged frames from a managed switch
# Create VLAN sub-interfaces on ether1 (trunk)

/interface vlan add name=vlan10 vlan-id=10 interface=ether1
/interface vlan add name=vlan20 vlan-id=20 interface=ether1
/interface vlan add name=vlan30 vlan-id=30 interface=ether1

# Assign IPv4 gateways
/ip address add address=10.1.10.1/24 interface=vlan10 comment="Corp VLAN 10"
/ip address add address=10.1.20.1/24 interface=vlan20 comment="Guest VLAN 20"
/ip address add address=10.1.30.1/24 interface=vlan30 comment="VoIP VLAN 30"
```

## DHCP Server per VLAN

```mikrotik
/ip pool add name=POOL-VLAN10 ranges=10.1.10.100-10.1.10.200
/ip pool add name=POOL-VLAN20 ranges=10.1.20.100-10.1.20.200

/ip dhcp-server add name=DHCP-VLAN10 interface=vlan10 address-pool=POOL-VLAN10 lease-time=12h
/ip dhcp-server add name=DHCP-VLAN20 interface=vlan20 address-pool=POOL-VLAN20 lease-time=1h

/ip dhcp-server network add address=10.1.10.0/24 gateway=10.1.10.1 dns-server=8.8.8.8
/ip dhcp-server network add address=10.1.20.0/24 gateway=10.1.20.1 dns-server=8.8.8.8
```

## Bridge with VLAN Filtering (RouterOS v7 Recommended)

```mikrotik
# Create bridge with VLAN filtering
/interface bridge add \
  name=bridge1 \
  vlan-filtering=yes \
  comment="Main switch bridge"

# Add trunk port (uplink to managed switch or server)
/interface bridge port add \
  interface=ether1 \
  bridge=bridge1 \
  frame-types=admit-only-vlan-tagged

# Add access ports
/interface bridge port add \
  interface=ether2 \
  bridge=bridge1 \
  pvid=10 \
  frame-types=admit-only-untagged-and-priority-tagged

/interface bridge port add \
  interface=ether3 \
  bridge=bridge1 \
  pvid=20 \
  frame-types=admit-only-untagged-and-priority-tagged

# Configure VLANs on bridge
/interface bridge vlan add \
  bridge=bridge1 \
  vlan-ids=10 \
  tagged=bridge1,ether1 \
  untagged=ether2

/interface bridge vlan add \
  bridge=bridge1 \
  vlan-ids=20 \
  tagged=bridge1,ether1 \
  untagged=ether3

# Add IPv4 addresses via VLAN interfaces on the bridge
/interface vlan add name=bridge1.10 vlan-id=10 interface=bridge1
/interface vlan add name=bridge1.20 vlan-id=20 interface=bridge1

/ip address add address=10.1.10.1/24 interface=bridge1.10
/ip address add address=10.1.20.1/24 interface=bridge1.20
```

## Verify VLANs

```mikrotik
# Show VLAN interfaces
/interface vlan print

# Show IP addresses on VLAN interfaces
/ip address print where interface~"vlan"

# Show bridge VLAN table
/interface bridge vlan print
```

## Conclusion

For router-on-a-stick, create VLAN sub-interfaces with `/interface vlan add` and assign IPv4 gateway addresses. For a full Layer 2 switch with VLAN filtering, use `/interface bridge` with `vlan-filtering=yes` and configure trunk/access ports via bridge VLAN settings. The bridge approach is more efficient for high-throughput switching.
