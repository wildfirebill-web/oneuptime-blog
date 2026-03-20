# How to Implement IPv6 Ingress Filtering (BCP 38/RFC 2827)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Security, BCP38, Ingress Filtering, Spoofing Prevention

Description: Learn how to implement BCP 38 ingress filtering for IPv6 to prevent source address spoofing and reduce the effectiveness of DDoS amplification attacks.

## Overview

BCP 38 (RFC 2827) defines ingress filtering: a network operator should drop packets arriving on an interface if the source address is not reachable via that interface. For IPv6 this prevents source address spoofing, which is a prerequisite for many DDoS amplification attacks. IPv6's lack of NAT makes source address validation more important, not less.

## Why IPv6 Needs BCP 38

IPv4 networks frequently hide behind NAT, which inadvertently validates source addresses. IPv6 uses no NAT — every host has a globally-routable address. Without ingress filtering, any host can spoof any IPv6 source address and launch:

- Amplification attacks using ICMPv6 echo to multicast groups
- NDP exhaustion with spoofed sources
- BGP session disruption via spoofed TCP packets
- Reflection attacks using spoofed source to misdirect responses

## Ingress Filtering Concepts

```mermaid
flowchart LR
    Customer[Customer\n2001:db8:cust::/48] --> ISP_Router[ISP Router]
    ISP_Router -- "Source 2001:db8:cust::/48\n→ ACCEPT" --> Internet
    ISP_Router -- "Source 2001:db8:other::/32\n→ DROP (spoofed)" --> Sink[/dev/null]
```

Rule: If a packet arrives from Customer, its source address MUST be within the Customer's assigned prefix. Any other source = spoofed → DROP.

## Implementation: Router ACL

### Cisco IOS

```
! Customer assigned: 2001:db8:cust::/48
ipv6 access-list INGRESS-CUSTOMER
  permit ipv6 2001:db8:cust::/48 any    ! Allow customer's legitimate addresses
  deny   ipv6 any any log               ! Drop everything else (spoofed)

interface GigabitEthernet0/0
  description "Customer link"
  ipv6 access-group INGRESS-CUSTOMER in

! Also block bogon sources at upstream interface
ipv6 access-list INGRESS-UPSTREAM
  deny   ipv6 ::/128 any
  deny   ipv6 ::1/128 any
  deny   ipv6 2001:db8::/32 any        ! Documentation prefix
  deny   ipv6 fc00::/7 any             ! ULA from internet
  deny   ipv6 fe80::/10 any            ! Link-local from internet
  permit ipv6 any any
```

### Juniper JunOS

```
# Filter applied to customer-facing interface
set firewall family inet6 filter INGRESS-CUSTOMER term allow-customer from source-address 2001:db8:cust::/48
set firewall family inet6 filter INGRESS-CUSTOMER term allow-customer then accept
set firewall family inet6 filter INGRESS-CUSTOMER term deny-spoof then reject

set interfaces ge-0/0/0 unit 0 family inet6 filter input INGRESS-CUSTOMER
```

### Linux iptables/nftables

```bash
# nftables: Ingress filtering for a hosted customer
nft add table ip6 ingress
nft add chain ip6 ingress input { type filter hook input priority -100\; }

# Allow only from customer's assigned prefix
nft add rule ip6 ingress input iifname "eth1" ip6 saddr 2001:db8:cust::/48 accept
nft add rule ip6 ingress input iifname "eth1" drop
```

## Unicast Reverse Path Forwarding (uRPF)

uRPF is an automated form of ingress filtering — the router checks that there is a route back to the source address via the incoming interface:

```
! Cisco: Enable strict uRPF on customer interface
interface GigabitEthernet0/0
  ipv6 verify unicast source reachable-via rx   ! Strict mode
  ! or
  ipv6 verify unicast source reachable-via any  ! Loose mode (less effective)
```

```bash
# Linux: Enable strict reverse path filtering
sysctl -w net.ipv6.conf.eth1.accept_source_route=0
# Note: IPv6 uRPF is handled differently on Linux — use ip rules and routing tables
# Add: route to customer prefix via customer interface in main routing table
ip -6 route add 2001:db8:cust::/48 dev eth1
# Then packets arriving on eth1 claiming to be from other prefixes have no reverse path
```

## Source Address Validation at Access Layer (SAVI)

SAVI (RFC 7039) implements ingress filtering at the switch port level:

```
! Cisco: SAVI on access switch
ipv6 snooping policy SAVI-POLICY
  limit address-count 10

interface GigabitEthernet0/1
  ipv6 snooping attach-policy SAVI-POLICY
  ! SAVI learns valid addresses via DHCPv6/SLAAC monitoring
  ! Drops packets with non-learned source addresses
```

## Bogon Prefix Ingress Filter

Always apply at internet-facing interfaces:

```bash
# ip6tables: Drop bogon sources at ingress
for prefix in "::/128" "::1/128" "::ffff:0:0/96" "64:ff9b::/96" "100::/64" \
  "2001::/23" "2001:db8::/32" "2002::/16" "fc00::/7" "fe80::/10"; do
  ip6tables -A INPUT -s "$prefix" -j DROP
done
```

## Measuring BCP 38 Compliance

Use the CAIDA Spoofer project tools to test your network:

```bash
# Download and run spoofer test
wget https://spoofer.caida.org/cgi-bin/spoofer.pl
chmod +x spoofer.pl
./spoofer.pl --protocol ipv6

# Report shows whether your ISP blocks spoofed IPv6 packets
```

## Summary

IPv6 ingress filtering (BCP 38) is implemented via ACLs on customer-facing interfaces that only permit traffic with source addresses matching the customer's assigned prefix, router-level uRPF (Cisco: `ipv6 verify unicast source reachable-via rx`), and switch-level SAVI for access-layer enforcement. Always combine with bogon prefix filtering at internet-facing interfaces. Test compliance with the CAIDA Spoofer project to verify your ISP drops spoofed IPv6 packets.
