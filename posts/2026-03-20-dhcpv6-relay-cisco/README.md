# How to Configure DHCPv6 Relay on Cisco Routers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Cisco, DHCPv6, Relay, IOS, IOS-XE, Networking

Description: Configure DHCPv6 relay on Cisco IOS and IOS-XE routers to forward DHCPv6 messages from clients to remote DHCPv6 servers.

## Basic DHCPv6 Relay on Cisco IOS/IOS-XE

```
! Enable IPv6 unicast routing (required)
ipv6 unicast-routing

! Configure the client-facing interface with relay destination
interface GigabitEthernet0/1
 description Client LAN
 ipv6 address 2001:db8:1::1/64
 ipv6 nd managed-config-flag    ! Tell clients to use DHCPv6
 ipv6 nd other-config-flag      ! Or use SLAAC + DHCPv6 for options only
 ipv6 dhcp relay destination 2001:db8::dhcp-server
 no shutdown
```

## Relay to Multiple Servers

```
! Forward DHCPv6 to two servers for redundancy
interface GigabitEthernet0/1
 ipv6 address 2001:db8:1::1/64
 ipv6 dhcp relay destination 2001:db8::dhcp1
 ipv6 dhcp relay destination 2001:db8::dhcp2
 no shutdown
```

## Relay with Source Interface Specification

```
! Specify source interface for relay messages (recommended)
interface GigabitEthernet0/1
 ipv6 address 2001:db8:1::1/64
 ipv6 dhcp relay destination 2001:db8::dhcp-server GigabitEthernet0/0
 ! GigabitEthernet0/0 is the server-facing interface
 no shutdown
```

## DHCPv6 Relay with Option 37 (Remote ID) and Option 18 (Interface ID)

```
! Enable vendor-specific relay options
ipv6 dhcp relay option vpn
ipv6 dhcp relay option enterprise-id

! VRF-aware relay (for multi-tenant environments)
interface GigabitEthernet0/1
 vrf forwarding Tenant1
 ipv6 address 2001:db8:1::1/64
 ipv6 dhcp relay destination 2001:db8::dhcp-server vrf Management
 no shutdown
```

## Cisco IOS-XR DHCPv6 Relay

```
! IOS-XR syntax for DHCPv6 relay (different from IOS)
dhcp ipv6
 interface GigabitEthernet0/0/0/1
  relay destination 2001:db8::dhcp-server
  relay option vpn-mode
  !
 !
!

! Apply relay on client interface
interface GigabitEthernet0/0/0/1
 ipv6 address 2001:db8:1::1/64
```

## Cisco NX-OS DHCPv6 Relay (Nexus)

```
! Enable DHCPv6 relay feature
feature dhcp

! Configure relay on VLAN SVI
interface Vlan100
 ipv6 address 2001:db8:1::1/64
 ipv6 dhcp relay address 2001:db8::dhcp-server

! Show relay status
show ipv6 dhcp relay statistics
show ipv6 dhcp relay binding
```

## Verification Commands

```
! IOS/IOS-XE verification
show ipv6 dhcp relay statistics
show ipv6 dhcp binding
show ipv6 dhcp interface GigabitEthernet0/1

! Check relay is configured
show running-config | include dhcp relay

! Debug relay messages (use with caution on production)
debug ipv6 dhcp

! Clear relay statistics
clear ipv6 dhcp relay statistics
```

## Troubleshooting

```
! Common issues:

! 1. No relay activity
show ipv6 dhcp relay statistics
! If counters not incrementing, check:
! - nd managed-config-flag is set on client interface
! - Clients are sending SOLICITs (check with debug)

! 2. Relay forwards but server doesn't respond
! Check server-facing routing
show ipv6 route 2001:db8::dhcp-server

! 3. Link-local as relay source (not recommended)
! Always use a global unicast address as relay source
interface GigabitEthernet0/1
 ipv6 dhcp relay source-interface Loopback0

! 4. Clients getting wrong configuration
show ipv6 dhcp relay binding | include Interface
```

## Conclusion

Cisco's DHCPv6 relay is configured with `ipv6 dhcp relay destination` on the client-facing interface. Use `ipv6 nd managed-config-flag` (M-bit) to instruct clients to use DHCPv6 for address assignment, or `other-config-flag` (O-bit) for stateless DHCPv6 (DNS servers, domain names only). Multiple destinations provide server redundancy. The `source-interface` parameter ensures relay messages originate from a reachable global unicast address. On NX-OS, enable the `feature dhcp` first before configuring relay.
