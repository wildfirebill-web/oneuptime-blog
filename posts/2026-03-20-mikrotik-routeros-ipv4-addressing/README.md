# How to Configure IPv4 Addressing on MikroTik RouterOS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MikroTik, RouterOS, IPv4, Network Configuration, Winbox, CLI

Description: Configure IPv4 addresses on MikroTik RouterOS interfaces using the CLI and Winbox, including adding, removing, and verifying IP addresses on physical and virtual interfaces.

## Introduction

MikroTik RouterOS uses a flat IP address model: IP addresses are assigned to interfaces directly. A single interface can hold multiple IP addresses. Configuration is done via the RouterOS CLI (SSH or serial) or Winbox GUI.

## Add an IPv4 Address via CLI

```mikrotik
# Add IP address to ether1

/ip address add address=192.168.1.1/24 interface=ether1 comment="LAN Gateway"

# Add IP to ether2 (WAN)
/ip address add address=203.0.113.2/30 interface=ether2 comment="WAN Uplink"

# Add secondary IP to same interface
/ip address add address=10.0.0.1/8 interface=ether1 comment="Secondary"
```

## List All Addresses

```mikrotik
/ip address print
# Output:
# Flags: X - disabled, I - invalid, D - dynamic
#  #   ADDRESS            NETWORK         INTERFACE
#  0   192.168.1.1/24     192.168.1.0     ether1
#  1   203.0.113.2/30     203.0.113.0     ether2
#  2   10.0.0.1/8         10.0.0.0        ether1

# Show detail
/ip address print detail
```

## Modify and Remove Addresses

```mikrotik
# Change an address (use the entry number from print)
/ip address set 0 address=192.168.10.1/24

# Disable an address without removing
/ip address disable 0

# Enable
/ip address enable 0

# Remove by number
/ip address remove 2

# Remove by comment
/ip address remove [find comment="Secondary"]
```

## VLAN Interface with IPv4

```mikrotik
# Create VLAN interface
/interface vlan add name=vlan10 vlan-id=10 interface=ether1

# Add IP to VLAN
/ip address add address=10.1.10.1/24 interface=vlan10
```

## Bridge Interface with IPv4

```mikrotik
# Create bridge (LAN switch)
/interface bridge add name=bridge-lan

# Add ports to bridge
/interface bridge port add bridge=bridge-lan interface=ether2
/interface bridge port add bridge=bridge-lan interface=ether3

# Assign IP to bridge
/ip address add address=192.168.1.1/24 interface=bridge-lan
```

## Winbox Equivalent

Navigate to:
- **IP > Addresses > [+]** to add
- Set Address (with CIDR prefix), Network (auto-filled), Interface
- Click **OK**

## Default Gateway

```mikrotik
# Add default route (gateway)
/ip route add dst-address=0.0.0.0/0 gateway=203.0.113.1 comment="Default GW"

# Verify
/ip route print
```

## Verify Connectivity

```mikrotik
# Ping from router
/ping 8.8.8.8 count=4

# Show interface status
/interface print

# Show ARP table
/ip arp print
```

## Conclusion

MikroTik RouterOS assigns IP addresses to interfaces with `/ip address add`. Multiple addresses per interface are fully supported. Combine with VLAN and bridge interfaces for complex LAN designs, and always set a default route via `/ip route add` for internet access.
