# How to Understand the IPv6 Minimum MTU of 1280 Bytes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, MTU, Minimum MTU, RFC 8200, Networking

Description: Understand why IPv6 mandates a minimum link MTU of 1280 bytes, how this differs from IPv4, and the practical implications for network design and tunnel configurations.

## Introduction

RFC 8200 mandates that every link carrying IPv6 traffic must support a minimum MTU of 1280 bytes. This is significantly higher than IPv4's theoretical minimum (68 bytes) and practical recommendations (576 bytes). The 1280-byte minimum was chosen to ensure that every IPv6 link can carry a minimally useful packet without requiring fragmentation, simplifying the network stack.

## Why 1280 Bytes?

The 1280-byte minimum was set to satisfy several constraints simultaneously:

```
Design requirements for the IPv6 minimum MTU:

1. Must be larger than IPv6 base header (40 bytes) by a useful amount
   → At 1280 bytes: 1280 - 40 = 1240 bytes of payload available
   → At the minimum, a single PMTU discovery probe fits without fragmentation

2. Must accommodate any possible extension header combination
   → A single Hop-by-Hop option: 8 bytes
   → Fragment header: 8 bytes
   → 1280 - 40 - 8 - 8 = 1224 bytes payload → still useful

3. Should be large enough for common upper-layer protocols
   → DNS over UDP: typical query/response < 512 bytes (easily fits)
   → ICMPv6 error messages include at minimum the first 1280 bytes of offending packet
   → NDP messages fit easily in 1280 bytes

4. Provides 2× the IPv4 practical minimum (576 bytes)
   → Signals that IPv6 paths should have reasonable capacity
```

## Comparison with IPv4 Minimums

```
IPv4:
  RFC 791: Minimum MTU = 68 bytes
  RFC 1191: Recommended minimum = 576 bytes
  Practice: Most networks use 1500 bytes (Ethernet)

IPv6:
  RFC 8200: Minimum MTU = 1280 bytes (MANDATORY)
  RFC 8201: Recommended PMTU = at least 1280 bytes
  Practice: Most networks use 1500 bytes (Ethernet)
  Tunnels: Often reduce to 1480 or less (overhead)
```

## Practical Impact on Network Design

```
Links where MTU challenges arise:

1. DSL PPPoE:
   PPPoE header: 8 bytes
   Effective IPv6 MTU: 1500 - 8 = 1492 bytes (still > 1280 ✓)

2. IPv6-in-IPv4 tunnel (6in4, RFC 4213):
   IPv4 header overhead: 20 bytes
   Available for IPv6: 1500 - 20 = 1480 bytes (still > 1280 ✓)

3. IPv6-in-IPv4 + GRE tunnel:
   IPv4 (20) + GRE (4) = 24 bytes overhead
   Available for IPv6: 1500 - 24 = 1476 bytes (still > 1280 ✓)

4. VPN with IPsec (worst case):
   IPv4 (20) + ESP (8) + IV (16) + ICV (16) + Pad (~12) ≈ 72 bytes
   Available for IPv6: 1500 - 72 = 1428 bytes (still > 1280 ✓)

5. Nested tunnels (problematic):
   IPsec ESP tunnel + GRE + MPLS: could approach or exceed overhead
   May require reduced outer link MTU of 9000+ (jumbo frames)
```

## Configuring MTU on IPv6 Interfaces

```bash
# Check current MTU on all interfaces
ip link show | grep -E "^[0-9]+:|mtu"

# Specifically check IPv6-relevant MTU
ip -6 link show

# Set MTU on an interface (must be ≥ 1280 for IPv6)
sudo ip link set eth0 mtu 1500

# For tunnel interfaces, set appropriate MTU
# IPv6 over IPv4 tunnel (accounts for outer IPv4 header)
sudo ip link set sit0 mtu 1480

# Check IPv6 MTU as seen by the kernel
cat /proc/sys/net/ipv6/conf/eth0/mtu

# Check if any interfaces have MTU < 1280 (would break IPv6)
ip link show | awk '/mtu/ {for(i=1;i<=NF;i++) if ($i=="mtu") print $(i+1), $2}' | \
    awk '{if ($1 < 1280) print "WARNING: " $2 " has MTU " $1 " < 1280 (IPv6 broken)"}'
```

## Handling Sub-1280-MTU Links

If a link truly cannot support 1280-byte packets (rare but possible in some IoT contexts):

```bash
# Option 1: Increase the physical link MTU (preferred)
# Configure the underlying link to support ≥ 1500 bytes

# Option 2: Use 6LoWPAN (IPv6 over Low-Power Wireless)
# 6LoWPAN (RFC 4944) provides header compression and fragmentation
# for IEEE 802.15.4 links with 127-byte frames
# The 6LoWPAN layer fragments and reassembles to/from 1280-byte IPv6 packets

# The source (6LoWPAN border router) handles fragmentation
# The IPv6 layer above never sees sub-1280 packets
```

## ICMPv6 Packet Too Big and the 1280-byte Rule

RFC 8200 requires that if a Packet Too Big message specifies an MTU less than 1280 bytes, the source should send packets of exactly 1280 bytes with a Fragment Header:

```python
def handle_packet_too_big(notified_mtu: int, original_packet: bytes) -> dict:
    """
    Handle an ICMPv6 Packet Too Big message.

    Args:
        notified_mtu: The MTU value in the PTB message
        original_packet: The packet that was too big
    """
    if notified_mtu < 1280:
        # RFC 8200 Section 5: If the notified MTU < 1280, use 1280 bytes
        # and include a Fragment Header to prevent further fragmentation issues
        effective_mtu = 1280
        must_fragment = True
        print(f"PTB MTU {notified_mtu} < 1280: Use 1280 bytes with Fragment Header")
    else:
        effective_mtu = notified_mtu
        must_fragment = False
        print(f"PTB MTU {notified_mtu}: Update PMTU cache to {effective_mtu}")

    return {
        "notified_mtu": notified_mtu,
        "effective_mtu": effective_mtu,
        "must_use_fragment_header": must_fragment,
    }

# Test
print(handle_packet_too_big(1200, b""))  # Below 1280 - use 1280 + Fragment Header
print(handle_packet_too_big(1400, b""))  # Normal case - use 1400
```

## Conclusion

IPv6's 1280-byte minimum MTU is a hard requirement that simplifies protocol design by providing a guaranteed floor for packet size. Every IPv6 implementation can assume that a 1280-byte packet will reach any destination without fragmentation at the path level. Tunnel configurations must account for encapsulation overhead to ensure they still support 1280-byte inner packets. When configuring IPv6 interfaces or tunnels, always verify that the configured MTU leaves sufficient room for the IPv6 stack to function correctly.
