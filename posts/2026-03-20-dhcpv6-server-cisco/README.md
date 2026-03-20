# How to Configure a DHCPv6 Server on Cisco IOS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: DHCPv6, IPv6, Cisco, IOS, Networking, Prefix Delegation, Router

Description: Learn how to configure a DHCPv6 server on Cisco IOS and IOS-XE routers to assign IPv6 addresses, deliver DNS options, and delegate prefixes to downstream devices.

---

Cisco IOS routers can act as DHCPv6 servers, allowing them to assign IPv6 addresses and deliver network configuration to clients on directly connected interfaces. This guide covers stateful DHCPv6, stateless options, prefix delegation, and verification commands.

---

## Prerequisites

- Cisco IOS 12.4(24)T or later, or IOS-XE 3.x+
- IPv6 unicast routing enabled
- Interface IPv6 addresses configured

---

## Enable IPv6 Routing

```cisco
ipv6 unicast-routing
ipv6 cef
```

---

## Basic Stateful DHCPv6 Pool

```cisco
! Define the DHCPv6 pool
ipv6 dhcp pool CLIENTS
 address prefix 2001:db8:1::/64 lifetime 86400 43200
 dns-server 2001:db8:ff::53
 domain-name corp.example.com
 import all

! Apply the pool to the client-facing interface
interface GigabitEthernet0/1
 ipv6 address 2001:db8:1::1/64
 ipv6 dhcp server CLIENTS
 ! M=1 O=1 in RA - tells clients to use DHCPv6
 ipv6 nd managed-config-flag
 ipv6 nd other-config-flag
```

---

## Stateless DHCPv6 (Options Only, SLAAC for Addresses)

```cisco
ipv6 dhcp pool OPTIONS-ONLY
 dns-server 2001:db8:ff::53
 domain-name corp.example.com

interface GigabitEthernet0/1
 ipv6 address 2001:db8:1::1/64
 ipv6 dhcp server OPTIONS-ONLY
 ! O=1 only - clients use SLAAC for addresses, DHCPv6 for options
 ipv6 nd other-config-flag
```

---

## DHCPv6 Prefix Delegation

```cisco
! Pool for delegating /56 prefixes from a /32 block
ipv6 dhcp pool PD-POOL
 prefix-delegation pool DELEGATED-PREFIXES lifetime 86400 43200
 dns-server 2001:db8:ff::53

ipv6 local pool DELEGATED-PREFIXES 2001:db8:100::/40 56

interface GigabitEthernet0/0
 ipv6 address 2001:db8:ff::1/64
 ipv6 dhcp server PD-POOL
```

---

## Address Exclusions

```cisco
ipv6 dhcp pool CLIENTS
 address prefix 2001:db8:1::/64 lifetime 86400 43200
 dns-server 2001:db8:ff::53

! Exclude specific addresses from the pool
! (Add static assignments for reserved IPs separately)
```

---

## Static Host Reservations

```cisco
ipv6 dhcp pool CLIENTS
 host 2001:db8:1::10/128
  duid 0001000128abc123001122334455
  address 2001:db8:1::10 infinite infinite
```

---

## Verification Commands

```cisco
! Show all DHCPv6 bindings (leases)
show ipv6 dhcp binding

! Show DHCPv6 pool details
show ipv6 dhcp pool

! Show interface DHCPv6 configuration
show ipv6 dhcp interface GigabitEthernet0/1

! Show DHCPv6 statistics (messages sent/received)
show ipv6 dhcp statistics

! Show conflicts
show ipv6 dhcp conflict

! Debug DHCPv6 (use carefully in production)
debug ipv6 dhcp detail
```

---

## Example Output

```text
Router# show ipv6 dhcp binding

Client: FE80::211:22FF:FE33:4455
  DUID: 0001000128ABC123001122334455
  Username : unassigned
  VRF : default
  Interface: GigabitEthernet0/1
  IA NA: IA ID 0x00000001, T1 43200, T2 69120
    Address: 2001:DB8:1::A
            preferred lifetime 86400, valid lifetime 86400
            expires at Mar 21 2026 10:00:00
```

---

## Troubleshooting

```cisco
! Check if DHCPv6 is listening on the interface
show ipv6 dhcp interface

! Check RA flags
show ipv6 interface GigabitEthernet0/1 | include managed

! Clear a specific binding
clear ipv6 dhcp binding 2001:db8:1::A

! Clear all bindings (use with caution)
clear ipv6 dhcp binding *
```

---

## Best Practices

1. **Always set RA flags** - `managed-config-flag` for stateful, `other-config-flag` for stateless
2. **Set appropriate lifetimes** - T1 = 50% of valid lifetime, T2 = 80%
3. **Use address exclusions** for routers, servers with static addresses
4. **Monitor bindings** regularly to detect address exhaustion
5. **Enable logging** for DHCPv6 events on production routers

---

## Conclusion

Cisco IOS provides a full-featured DHCPv6 server capable of stateful address assignment, stateless option delivery, and prefix delegation. Configure the pool, apply it to the interface with correct RA flags, and use `show ipv6 dhcp binding` to verify client assignments.

---

*Monitor your Cisco network and IPv6 infrastructure with [OneUptime](https://oneuptime.com).*
