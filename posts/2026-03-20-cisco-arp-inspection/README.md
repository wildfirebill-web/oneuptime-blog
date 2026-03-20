# How to Configure ARP Inspection on Cisco Switches

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Networking, ARP, Cisco, Security, Switching

Description: Learn how to configure Dynamic ARP Inspection (DAI) on Cisco IOS switches to validate ARP packets and prevent ARP spoofing.

## What DAI Does

Dynamic ARP Inspection (DAI) on Cisco switches:
- Intercepts ARP packets on untrusted switch ports
- Validates IP-MAC bindings against the DHCP snooping binding database
- Drops ARP packets with invalid or suspicious mappings
- Logs violations for security auditing

## Prerequisites

DAI relies on DHCP snooping bindings. Enable DHCP snooping first:

```cisco
ip dhcp snooping
ip dhcp snooping vlan 10,20,30
no ip dhcp snooping information option

! Trust the uplink toward the DHCP server
interface GigabitEthernet0/24
 ip dhcp snooping trust
```

## Enable DAI on VLANs

```cisco
! Enable DAI on specific VLANs
ip arp inspection vlan 10
ip arp inspection vlan 20,30

! Verify
show ip arp inspection vlan 10
```

## Configure Trusted Ports

Ports connected to routers, other switches, or servers need to be trusted:

```cisco
! Trust uplink and server ports
interface GigabitEthernet0/24
 ip arp inspection trust

interface GigabitEthernet0/23
 ip arp inspection trust
```

Access ports facing end users should remain untrusted (default).

## Add ARP ACLs for Static IP Hosts

Hosts using static IPs are not in the DHCP snooping binding table, so you need ARP ACLs:

```cisco
! Define static IP-to-MAC bindings
arp access-list STATIC_SERVERS
 permit ip host 10.0.0.10 mac host aa:bb:cc:dd:ee:01
 permit ip host 10.0.0.20 mac host 00:11:22:33:44:55
 permit ip host 10.0.0.1 mac host ff:ee:dd:cc:bb:aa

! Apply to VLAN 10
ip arp inspection filter STATIC_SERVERS vlan 10
```

## Enable Enhanced Validations

```cisco
! Validate source MAC, destination MAC, and sender IP
ip arp inspection validate src-mac
ip arp inspection validate dst-mac
ip arp inspection validate ip

! Enable all three at once
ip arp inspection validate src-mac dst-mac ip
```

| Validation | What It Checks |
|------------|----------------|
| `src-mac` | ARP sender MAC matches Ethernet source MAC |
| `dst-mac` | ARP target MAC matches Ethernet destination MAC (for replies) |
| `ip` | Sender IP is not 0.0.0.0, broadcast, or multicast |

## Configure ARP Rate Limiting

```cisco
! Limit untrusted ports to 100 ARP packets per second
interface range GigabitEthernet0/1 - 20
 ip arp inspection limit rate 100
 ip arp inspection limit rate 100 burst interval 2
```

Enable automatic recovery if a port is put into error-disabled state:

```cisco
errdisable recovery cause arp-inspection
errdisable recovery interval 300
```

## Monitoring and Verification

```cisco
! Show DAI configuration
show ip arp inspection

! Show per-VLAN statistics
show ip arp inspection statistics vlan 10

! Show per-interface statistics
show ip arp inspection interfaces

! Clear statistics
clear ip arp inspection statistics
```

Sample output:

```text
 Vlan      Forwarded        Dropped   DHCP Drops   ACL Drops
 ----      ---------        -------   ----------   ---------
   10          34521            312            8         304
```

## Full Configuration Example

```cisco
! DHCP Snooping base
ip dhcp snooping
ip dhcp snooping vlan 10
no ip dhcp snooping information option

! DAI on VLAN 10
ip arp inspection vlan 10
ip arp inspection validate src-mac dst-mac ip

! ARP ACL for static hosts
arp access-list STATIC_HOSTS
 permit ip host 10.0.0.1 mac host ff:ee:dd:cc:bb:aa

ip arp inspection filter STATIC_HOSTS vlan 10

! Rate limiting
interface range GigabitEthernet0/1 - 22
 ip arp inspection limit rate 100

! Trust uplink
interface GigabitEthernet0/24
 ip dhcp snooping trust
 ip arp inspection trust
```

## Key Takeaways

- DAI validates ARP packets on untrusted ports against the DHCP snooping binding table.
- Trusted ports bypass DAI (use for uplinks and servers).
- ARP ACLs are required for static IP hosts not covered by DHCP snooping.
- Enhanced validations (`src-mac`, `dst-mac`, `ip`) catch additional attack vectors.

**Related Reading:**

- [How to Prevent ARP Poisoning with Dynamic ARP Inspection](https://oneuptime.com/blog/post/2026-03-20-prevent-arp-poisoning-dai/view)
- [How to Detect ARP Spoofing Attacks on Your Network](https://oneuptime.com/blog/post/2026-03-20-arp-spoofing-detection-scapy-ipv4/view)
