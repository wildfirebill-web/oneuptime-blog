# How to Configure IPv6 on Cisco IOS-XE Routers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Cisco, IOS-XE, Router, Networking, Configuration

Description: Configure IPv6 on Cisco IOS-XE routers with modern features including segment routing, RDNSS in RA, and YANG/RESTCONF-based configuration.

## Introduction

Cisco IOS-XE builds on IOS with a modular architecture running on Cisco's hardware platforms (ASR, ISR 4000, Catalyst 8000). While most IPv6 commands are backward compatible with IOS, IOS-XE adds modern features like model-driven programmability, RDNSS in RA, and improved IPv6 segment routing support.

## Step 1: Enable IPv6 and Basic Interface Configuration

```
! Enable IPv6 unicast routing
Router(config)# ipv6 unicast-routing
Router(config)# ipv6 cef

! Configure LAN interface
Router(config)# interface GigabitEthernet0/0/0
Router(config-if)# description LAN Interface
Router(config-if)# ipv6 address 2001:db8:1:1::1/64
Router(config-if)# ipv6 enable
Router(config-if)# no shutdown

! Configure WAN interface
Router(config)# interface GigabitEthernet0/0/1
Router(config-if)# description WAN Interface - ISP
Router(config-if)# ipv6 address autoconfig   ! Or static if ISP provides one
Router(config-if)# no shutdown
```

## Step 2: Configure RA with RDNSS (IOS-XE 16.6+)

IOS-XE 16.6 and later support RFC 8106 RDNSS directly in RA:

```
! Advertise DNS servers via RA on the LAN interface
Router(config)# interface GigabitEthernet0/0/0
Router(config-if)# ipv6 nd ra dns server 2001:db8:1:1::53 infinite
Router(config-if)# ipv6 nd ra dns server 2606:4700:4700::1111 infinite

! Advertise DNS search list (IOS-XE 16.9+)
Router(config-if)# ipv6 nd ra dns search-list corp.example.com infinite
Router(config-if)# ipv6 nd ra dns search-list example.com infinite

! Verify
Router# show ipv6 nd ra dns GigabitEthernet0/0/0
```

## Step 3: Configure DHCPv6 with Prefix Delegation

IOS-XE handles DHCPv6-PD for both server and client roles:

```
! DHCPv6 Pool for assigning prefixes to downstream routers
Router(config)# ipv6 dhcp pool PD-POOL
Router(config-dhcpv6)# prefix-delegation pool PD-PREFIXES lifetime 86400 14400
Router(config-dhcpv6)# dns-server 2001:db8:1:1::53
Router(config-dhcpv6)# domain-name example.com

! Create the prefix delegation pool
Router(config)# ipv6 local pool PD-PREFIXES 2001:db8::/48 56

! Apply to WAN interface (server mode)
Router(config)# interface GigabitEthernet0/0/1
Router(config-if)# ipv6 dhcp server PD-POOL

! Or configure as DHCPv6-PD client (requesting prefix from ISP)
Router(config)# interface GigabitEthernet0/0/1
Router(config-if)# ipv6 dhcp client pd PREFIX hint ::/48
```

## Step 4: Configure OSPF v3 with Address Families (IOS-XE)

IOS-XE supports OSPFv3 with address family syntax (newer approach):

```
! Modern OSPFv3 with address family
Router(config)# router ospfv3 1
Router(config-router)# !
Router(config-router)# address-family ipv6 unicast
Router(config-router-af)# router-id 1.1.1.1

! Enable on interfaces using the address-family syntax
Router(config)# interface GigabitEthernet0/0/0
Router(config-if)# ospfv3 1 ipv6 area 0
```

## Step 5: Configure Using RESTCONF (Model-Driven)

IOS-XE supports RESTCONF for programmatic configuration:

```bash
# Enable RESTCONF on the router
# Router(config)# restconf
# Router(config)# ip http server
# Router(config)# ip http secure-server

# Configure IPv6 address via RESTCONF (from a management host)
curl -s -X PATCH \
    -H "Content-Type: application/yang-data+json" \
    -H "Accept: application/yang-data+json" \
    -u admin:password \
    "https://router/restconf/data/ietf-interfaces:interfaces/interface=GigabitEthernet0%2F0%2F0" \
    -d '{
        "ietf-interfaces:interface": {
            "name": "GigabitEthernet0/0/0",
            "ietf-ip:ipv6": {
                "address": [{
                    "ip": "2001:db8:1:1::1",
                    "prefix-length": 64
                }]
            }
        }
    }'
```

## Verification Commands

```
! Show IPv6 interfaces and addresses
Router# show ipv6 interface brief

! Show IPv6 routing table
Router# show ipv6 route

! Show DHCPv6 bindings and pools
Router# show ipv6 dhcp pool
Router# show ipv6 dhcp binding

! Show RDNSS configuration
Router# show ipv6 nd ra dns GigabitEthernet0/0/0

! Show OSPFv3 status
Router# show ospfv3 neighbor
Router# show ospfv3 database

! Ping test
Router# ping 2606:4700:4700::1111 source GigabitEthernet0/0/0
```

## Conclusion

Cisco IOS-XE extends IOS IPv6 capabilities with modern features like RDNSS in RA (IOS-XE 16.6+), model-driven programmability via RESTCONF, and improved DHCPv6-PD handling. The core IPv6 configuration syntax remains compatible with IOS, making migration straightforward. Use IOS-XE's RESTCONF interface for automated IPv6 provisioning in DevOps workflows.
