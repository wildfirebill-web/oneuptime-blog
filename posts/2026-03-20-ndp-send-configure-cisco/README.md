# How to Configure SEND on Cisco Routers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: SEND, Cisco, NDP Security, IPv6 Security, IOS

Description: Configure Secure Neighbor Discovery (SEND) on Cisco IOS routers, including CGA setup and RSA key generation for NDP authentication.

## Introduction

Cisco IOS has limited SEND support, primarily in research and specialized platforms. Cisco's mainstream security solution for NDP protection is RA Guard, DHCPv6 Guard, and IPv6 Source Guard through the "IPv6 First Hop Security" feature set. This guide covers what Cisco SEND support exists and the more practical First Hop Security alternatives.

## Cisco SEND Support Status

```
Cisco SEND implementation:
  - Research-level support on some platforms
  - Not available in standard IOS releases (as of most IOS versions)
  - Cisco recommends First Hop Security (FHS) for production deployments
  - FHS provides RA Guard, DHCPv6 Guard, ND Inspection, and Source Guard

For production security: use First Hop Security (not SEND)
```

## Cisco IPv6 First Hop Security (Practical Alternative)

```
Cisco IOS IPv6 First Hop Security commands:

RA Guard (prevents rogue Router Advertisements):
  ipv6 nd raguard policy HOST_POLICY
    device-role host    ← This port is NOT a router
  interface GigabitEthernet0/1
    ipv6 nd raguard attach-policy HOST_POLICY

DHCPv6 Guard (prevents rogue DHCP servers):
  ipv6 dhcp guard policy HOST_DHCP
    device-role client
  interface GigabitEthernet0/1
    ipv6 dhcp guard attach-policy HOST_DHCP

Binding Table (tracks address bindings):
  ipv6 neighbor binding max-entries 1000

IPv6 Source Guard:
  ipv6 source-guard attach-policy my-policy
```

## Configuring RA Guard on Cisco

```
! Create RA Guard policy for router ports
ipv6 nd raguard policy ROUTER_POLICY
 device-role router
 trusted-port

! Create RA Guard policy for host ports
ipv6 nd raguard policy HOST_POLICY
 device-role host
 !  ← No trusted-port: RA from this port will be blocked

! Apply to interfaces
interface GigabitEthernet0/1
 description Access port - host connected
 ipv6 nd raguard attach-policy HOST_POLICY

interface GigabitEthernet0/24
 description Uplink to router
 ipv6 nd raguard attach-policy ROUTER_POLICY

! Verify RA Guard status
show ipv6 nd raguard policy
show ipv6 nd raguard interface GigabitEthernet0/1
```

## Configuring Complete First Hop Security on Cisco

```
! Step 1: Enable IPv6 snooping (builds binding table)
ipv6 snooping policy SNOOPING_POLICY
 security-level guard

vlan configuration 10
 ipv6 snooping attach-policy SNOOPING_POLICY

! Step 2: RA Guard (block rogue RAs)
ipv6 nd raguard policy RA_GUARD_HOST
 device-role host

interface range GigabitEthernet0/1-23
 ipv6 nd raguard attach-policy RA_GUARD_HOST

! Step 3: DHCPv6 Guard
ipv6 dhcp guard policy DHCP_GUARD_HOST
 device-role client

interface range GigabitEthernet0/1-23
 ipv6 dhcp guard attach-policy DHCP_GUARD_HOST

! Step 4: Source Guard (only allows IPs from binding table)
ipv6 source-guard attach-policy SOURCE_GUARD_POLICY

! Verify binding table
show ipv6 neighbor binding
```

## RSA Key Generation (for any SEND-capable platform)

```
! Generate RSA keys for SEND (if platform supports it)
crypto key generate rsa general-keys modulus 2048 label SEND_KEY

! Show generated keys
show crypto key mypubkey rsa SEND_KEY

! For SEND: these keys would be used to generate CGAs
! (platform-specific commands when SEND is supported)
```

## Conclusion

Cisco's practical NDP security solution is IPv6 First Hop Security, not SEND. RA Guard blocks rogue Router Advertisements on host-facing ports. DHCPv6 Guard blocks unauthorized DHCP servers. IPv6 Snooping builds a binding table of legitimate address-to-port mappings. IPv6 Source Guard uses the binding table to block spoofed source addresses. These four features together provide defense-in-depth for IPv6 first-hop security without the complexity and limited vendor support of SEND.
