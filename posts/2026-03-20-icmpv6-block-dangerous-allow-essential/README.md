# How to Block Dangerous ICMPv6 While Allowing Essential Messages

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ICMPv6, Security, Firewall, IPv6, NDP Guard

Description: Identify dangerous ICMPv6 message types, implement firewall rules to block rogue RA and NDP attacks, and protect the network while keeping essential ICMPv6 functional.

## Introduction

While essential ICMPv6 must never be blocked, some ICMPv6 messages can be weaponized for attacks. Rogue Router Advertisements can redirect traffic or assign attacker-controlled addresses. Rogue Neighbor Advertisements can poison the neighbor cache. The challenge is to block illegitimate ICMPv6 from untrusted sources while allowing legitimate messages from trusted routers.

## Dangerous ICMPv6 Attack Vectors

```
ICMPv6-based attacks:

1. Rogue Router Advertisement (Type 134) attack:
   Attacker sends RA with:
   - Attacker's link-local as default gateway
   - Short prefix lifetime (forcing re-SLAAC)
   - M/O flags set (redirecting to rogue DHCPv6)
   Impact: Traffic hijacking, MITM, denial of service

2. Rogue Neighbor Advertisement (Type 136) attack:
   Attacker sends NA claiming to own a neighbor's IPv6 address
   Impact: Neighbor cache poisoning (IPv6 equivalent of ARP poisoning)

3. Neighbor Solicitation flooding (Type 135):
   Flood of NS packets to non-existent addresses
   Forces target to process and respond to all
   Impact: CPU exhaustion, neighbor cache overflow

4. ICMPv6 redirect attack (Type 137):
   Attacker sends Redirect to change a host's path to a destination
   Impact: Traffic hijacking, route manipulation

5. MLD query spoofing (Type 130):
   Forged MLD queries to suppress multicast listeners
   Impact: Multicast disruption
```

## Blocking Rogue Router Advertisements

```bash
# Only allow RA from specific trusted routers (by source address)
# Block all other RAs

# Allow RA from your legitimate router's link-local address
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type router-advertisement \
    -s fe80::1 -j ACCEPT

# Block all other Router Advertisements
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type router-advertisement \
    -j DROP

# This prevents rogue RA attacks from unauthorized devices
# Note: This requires knowing your router's link-local address

# For multiple routers:
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type router-advertisement \
    -s fe80::1 -j ACCEPT
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type router-advertisement \
    -s fe80::2 -j ACCEPT
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type router-advertisement \
    -j DROP
```

## Using RA Guard (Switchport Feature)

RA Guard is the preferred solution for blocking rogue RAs at the switch layer:

```bash
# Cisco switch: RA Guard configuration (blocks RA on access ports)
# interface GigabitEthernet0/1
#   ipv6 nd raguard attach-policy HOST_POLICY

# On Linux (for a bridge/router acting as switch):
# Install radvd-check or use ebtables to block RA on untrusted ports

# ebtables: block RA on a specific bridge port (untrusted)
# ebtables -A FORWARD -p IPv6 --ip6-proto 58 --ip6-icmp-type 134 \
#     -i eth1 -j DROP

# Alternative: ra-guard with ip6tables + source constraints
# Allow only RA from link-local addresses of trusted routers
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type 134 \
    -m mac --mac-source 00:11:22:33:44:55 -j ACCEPT
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type 134 -j DROP
```

## Protecting Against Neighbor Cache Poisoning

```bash
# Limit NA rate (prevents flooding/poisoning attacks)
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type neighbor-advertisement \
    -m limit --limit 100/second --limit-burst 200 -j ACCEPT
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type neighbor-advertisement \
    -j DROP  # Drop excess NA (rate limit exceeded)

# Limit NS rate (prevents NS flooding)
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type neighbor-solicitation \
    -m limit --limit 200/second --limit-burst 400 -j ACCEPT
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type neighbor-solicitation \
    -j DROP

# Block Redirect from untrusted sources
# Only allow Redirect from the current default gateway
DEFAULT_GW=$(ip -6 route show default | awk '/via/ {print $3; exit}')
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type redirect \
    -s "$DEFAULT_GW" -j ACCEPT
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type redirect -j DROP
```

## SEND (Secure Neighbor Discovery)

```
SEND (RFC 3971) provides cryptographic authentication for NDP:

- Uses RSA keys to sign NS/NA/RS/RA messages
- Routers and hosts have cryptographically generated addresses (CGA)
- Prevents rogue RA and NA attacks at the protocol level
- Requires: PKI infrastructure, all NDP-speaking devices support SEND
- Practical limitation: rarely deployed (complex; vendor support limited)
- Alternative: RA Guard + DHCP snooping at switch layer (more practical)
```

## Conclusion

The most dangerous ICMPv6 messages are Rogue Router Advertisements (Type 134) and Rogue Neighbor Advertisements (Type 136). The defense strategy: restrict RA acceptance to known router link-local addresses, apply rate limits to NA and NS flooding, and block Redirect messages from untrusted sources. RA Guard at the switch layer is the most operationally practical solution for rogue RA prevention. Rate limiting NDP messages with ip6tables provides a software fallback when switch-level RA Guard is not available. All these measures must be applied without blocking legitimate ICMPv6 from trusted sources.
