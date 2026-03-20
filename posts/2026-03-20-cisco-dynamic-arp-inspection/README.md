# How to Configure Dynamic ARP Inspection on Cisco Switches

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Cisco, Dynamic ARP Inspection, DAI, IPv4, Security, ARP Spoofing, IOS

Description: Configure Dynamic ARP Inspection (DAI) on Cisco IOS switches to prevent ARP spoofing and poisoning attacks by validating ARP packets against the DHCP snooping binding table.

## Introduction

Dynamic ARP Inspection intercepts ARP requests and replies on untrusted ports and validates them against the DHCP snooping binding table. An ARP message claiming a MAC-to-IP mapping not in the binding table is dropped, preventing ARP spoofing and man-in-the-middle attacks.

## Prerequisites

DHCP snooping must be enabled and have a populated binding table before DAI is effective.

## Enable DAI

```cisco
! Enable DHCP snooping (prerequisite)
ip dhcp snooping
ip dhcp snooping vlan 10,20

! Enable DAI per VLAN
ip arp inspection vlan 10,20

! Trust the uplink — ARP from trusted ports is not validated
interface GigabitEthernet0/24
 description Uplink-to-Distribution
 ip arp inspection trust

! Access ports are untrusted by default
```

## DAI Rate Limiting

```cisco
! Limit ARP packets on untrusted ports
interface range GigabitEthernet0/1 - 20
 ip arp inspection limit rate 100 burst interval 1
 ! 100 ARP packets per second; port goes err-disabled if exceeded
```

## ARP ACL for Static IP Hosts (Servers without DHCP)

```cisco
! Servers with static IPs won't have a DHCP snooping binding
! Use an ARP ACL to explicitly permit their ARP messages

arp access-list STATIC-SERVERS
 permit ip host 10.1.20.10 mac host 00:1a:2b:3c:4d:5e
 permit ip host 10.1.20.11 mac host 00:aa:bb:cc:dd:ee

ip arp inspection filter STATIC-SERVERS vlan 20
```

## Additional Validation Checks

```cisco
! Enable extra validation (optional but recommended)
ip arp inspection validate src-mac dst-mac ip

! src-mac: sender MAC in ARP must match Ethernet source MAC
! dst-mac: target MAC in ARP reply must match Ethernet dest MAC
! ip:      block ARP with invalid IP (0.0.0.0, broadcast, multicast)
```

## Verify DAI

```cisco
! Show DAI status per VLAN
show ip arp inspection vlan 10

! Show interfaces and trust state
show ip arp inspection interfaces

! Show statistics (forwarded/dropped counts)
show ip arp inspection statistics

! Example output:
! Vlan  Forwarded  Dropped  DHCP Drops  ACL Drops  ...
! ----  ---------  -------  ----------  ---------
!   10       1523       12           4          8
```

## Recover err-disabled Ports

```cisco
! Automatic recovery for DAI violations
errdisable recovery cause arp-inspection
errdisable recovery interval 300

! Manual recovery
interface GigabitEthernet0/5
 shutdown
 no shutdown
```

## Conclusion

Dynamic ARP Inspection stops ARP spoofing by cross-referencing ARP messages against the DHCP snooping binding table. Trust uplink ports, rate-limit access ports to prevent ARP floods, and add static ARP ACLs for hosts with manual IP configurations. Enable `ip arp inspection validate src-mac dst-mac ip` for the most thorough validation.
