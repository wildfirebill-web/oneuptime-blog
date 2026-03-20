# How SLAAC Handles Multiple Prefixes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: SLAAC, Multiple Prefixes, IPv6, Address Selection, Multihoming

Description: Understand how SLAAC handles multiple prefix advertisements, how hosts generate multiple addresses, and how source address selection works when multiple SLAAC addresses exist.

## Introduction

When a router advertises multiple prefixes in its Router Advertisements, SLAAC-capable hosts generate one address per prefix. Multiple prefixes arise in dual-stack environments, ISP prefix delegation scenarios, IPv6 renumbering, and multihomed environments. Understanding how hosts manage multiple SLAAC addresses and which address is chosen for outbound connections is essential for troubleshooting IPv6 connectivity.

## How Multiple Prefixes Arise

```
Scenarios with Multiple IPv6 Prefixes:

1. Renumbering (old + new prefix):
   Old prefix: 2001:db8:old::/64 (being phased out)
   New prefix: 2001:db8:new::/64 (being introduced)
   → Both advertised simultaneously during transition
   → Hosts have two global addresses

2. Multiple upstream ISPs (multihoming):
   ISP A prefix: 2001:db8:a::/64
   ISP B prefix: 2001:db8:b::/64
   → Both prefixes advertised via separate RAs
   → Hosts have two global addresses

3. GUA + ULA (common enterprise pattern):
   Global prefix: 2001:db8::/64 (public, internet-routable)
   ULA prefix:    fd00::/64 (private, local use only)
   → Both advertised in RA
   → Hosts have one GUA + one ULA address

4. Multiple /64s from prefix delegation:
   ISP assigns /48, router sub-delegates:
   Prefix A: 2001:db8:1::/64 (VLAN 10)
   Prefix B: 2001:db8:2::/64 (VLAN 20)
   (Each VLAN gets its own prefix, not multiple prefixes per host)
```

## SLAAC with Multiple Prefixes

```bash
# Router advertising two prefixes (radvd config):
# interface eth1 {
#     AdvSendAdvert on;
#     prefix 2001:db8:a::/64 {
#         AdvOnLink on; AdvAutonomous on;
#         AdvValidLifetime 2592000; AdvPreferredLifetime 604800;
#     };
#     prefix 2001:db8:b::/64 {
#         AdvOnLink on; AdvAutonomous on;
#         AdvValidLifetime 2592000; AdvPreferredLifetime 604800;
#     };
# };

# On a SLAAC host receiving both prefixes:
ip -6 addr show eth0
# inet6 2001:db8:a::211:22ff:fe33:4455/64 scope global dynamic
#    valid_lft 2591900sec preferred_lft 604700sec
# inet6 2001:db8:b::211:22ff:fe33:4455/64 scope global dynamic
#    valid_lft 2591900sec preferred_lft 604700sec
# inet6 fe80::211:22ff:fe33:4455/64 scope link
#    valid_lft forever preferred_lft forever

# Both global addresses exist simultaneously
# Same interface identifier (EUI-64) for both
# With privacy extensions: random IID per prefix (different IID for each)
```

## Source Address Selection with Multiple Prefixes

```
RFC 6724 Source Address Selection Rules:

When multiple source addresses exist for an outbound connection,
the kernel applies these rules in order:

Rule 1: Prefer same address as destination
Rule 2: Prefer appropriate scope
        (link-local for link-local dst, global for global dst)
Rule 3: Avoid deprecated addresses
Rule 4: Prefer home address (Mobile IPv6)
Rule 5: Prefer outgoing interface
Rule 6: Prefer matching label
        (uses Policy Table to match src prefix to dst prefix)
Rule 7: Prefer temporary addresses (privacy extensions)
Rule 8: Use longest matching prefix

For multiple GUA prefixes without specific policy:
  - Rule 8 (longest match) often determines selection
  - If equal prefix length: implementation-specific
  - May be round-robin or first-configured wins
```

## Policy Table for Source Address Selection

```bash
# View current source address selection policy table
ip -6 rule show
# (Note: ip rule and ip -6 rule show routing policy, not address selection)

# Source address selection policy table (RFC 6724 Section 2.1)
# Kernel hardcodes the default policy table:
# Prefix        Precedence  Label
# ::1/128       50          0      (loopback)
# ::/0          40          1      (global)
# ::ffff:0:0/96 35          4      (IPv4-mapped)
# 2002::/16     30          2      (6to4)
# 2001::/32     5           5      (Teredo)
# fc00::/7      3           13     (ULA)
# ::/96         1           3      (IPv4 compatible, deprecated)
# fec0::/10     1           11     (site-local, deprecated)
# 3ffe::/16     1           12     (6bone, deprecated)

# The "label" is used in Rule 6:
# Source and destination in same label = preferred match

# ULA (fc00::/7, label 13) prefers ULA destinations
# Global (label 1) prefers global destinations
# This prevents ULA source from being used for global destinations
```

## Testing Multiple Address Source Selection

```bash
# Force a specific source address for testing
curl -6 --interface 2001:db8:a::211:22ff:fe33:4455 https://example.com

# Or use -6 source address selection test:
# Create a connection and check source
python3 -c "
import socket
s = socket.socket(socket.AF_INET6, socket.SOCK_DGRAM)
s.connect(('2001:4860:4860::8888', 53))
print('Source address:', s.getsockname()[0])
s.close()
"

# Use ip to manipulate source address selection
# Add a preferred source address for specific destinations
sudo ip -6 rule add from 2001:db8:a::/64 lookup 100
sudo ip -6 route add default via fe80::1 dev eth0 table 100
# Packets sourced from a:: use table 100 (specific routing)
```

## ULA + GUA Coexistence

```bash
# Common pattern: ULA for internal traffic, GUA for internet
# Router advertises:
#   Global: 2001:db8::/64 (internet routing)
#   ULA:    fd00::/64 (internal only)

# Host addresses:
ip -6 addr show eth0
# inet6 2001:db8::211:22ff:fe33:4455/64 scope global  ← GUA
# inet6 fd00::211:22ff:fe33:4455/64 scope global      ← ULA

# Source address selection:
# For destination on internet (2001:...):
#   ULA source: would work (router NAT66 or not routable externally)
#   GUA source: preferred (internet-routable)
#   → RFC 6724 Rule 2 (scope) + label table → prefers GUA for global dest

# For destination on local network (fd00::...):
#   ULA source: preferred (same label in policy table = label 13)
#   GUA source: different label → less preferred for ULA dest
#   → RFC 6724 Rule 6 (label matching) → prefers ULA for ULA dest

# Verify which source is used for internal vs external
python3 -c "
import socket
# Test internet destination
s = socket.socket(socket.AF_INET6, socket.SOCK_DGRAM)
s.connect(('2001:4860:4860::8888', 53))
print('Internet source:', s.getsockname()[0])
s.close()
# Test internal destination
s = socket.socket(socket.AF_INET6, socket.SOCK_DGRAM)
s.connect(('fd00::1', 53))
print('Internal source:', s.getsockname()[0])
s.close()
"
```

## Conclusion

SLAAC hosts generate one address per advertised prefix, resulting in multiple global unicast addresses when multiple prefixes are advertised. Source address selection (RFC 6724) uses an 8-rule priority system to choose among available source addresses. Key rules are: avoid deprecated addresses (Rule 3), prefer temporary addresses (Rule 7), and match the policy label (Rule 6). The ULA + GUA pattern relies on label matching (Rule 6) to ensure ULA sources are preferred for internal destinations and GUA sources for internet destinations. Understanding multiple-prefix behavior is essential for troubleshooting connectivity in multihomed or renumbering scenarios.
