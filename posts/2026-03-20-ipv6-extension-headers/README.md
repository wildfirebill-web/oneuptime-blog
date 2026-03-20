# How to Understand IPv6 Extension Headers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Extension Headers, Networking, Protocol, RFC 8200

Description: Understand IPv6 extension headers, their purpose, how they are chained together, and which ones are processed by routers versus only by endpoints.

## Introduction

IPv6 extension headers are optional headers placed between the IPv6 base header and the upper-layer protocol header. They provide a flexible mechanism for adding features (routing, fragmentation, security, mobility) without modifying the base header. Extension headers are chained using the Next Header field — each header points to the next one in the chain.

## Extension Header Chain Structure

```
IPv6 Base Header (40 bytes)
  Next Header = 0 → Hop-by-Hop Options Header
                ↓
  Hop-by-Hop Options Header
    Next Header = 43 → Routing Header
                   ↓
  Routing Header
    Next Header = 44 → Fragment Header
                   ↓
  Fragment Header
    Next Header = 6 → TCP
                  ↓
  TCP Segment + Application Data
```

## All Defined Extension Headers

| Next Header Value | Extension Header | Processed By |
|---|---|---|
| 0 | Hop-by-Hop Options | All routers + destination |
| 43 | Routing Header | Routers in the path + destination |
| 44 | Fragment Header | Destination only |
| 50 | ESP (IPsec) | Destination only |
| 51 | AH (IPsec Auth) | Destination only |
| 59 | No Next Header | — |
| 60 | Destination Options | Destination only |
| 135 | Mobility Header | Destination only |
| 139 | HIP (Host Identity) | Destination only |
| 140 | Shim6 | Destination only |

## Common Extension Header Format

Most extension headers (except Fragment) share a common format:

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Next Header  |  Hdr Ext Len  |                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+                               +
|                   Header-specific data                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

Next Header:  Identifies the next header in the chain
Hdr Ext Len:  Length of this header in 8-byte units, NOT including the first 8 bytes
              Total header length = (Hdr Ext Len + 1) × 8 bytes
```

## Parsing Extension Headers

```python
import struct

# Extension header type codes
EXT_HEADERS = {
    0:   ("Hop-by-Hop Options", "variable"),
    43:  ("Routing Header", "variable"),
    44:  ("Fragment Header", "fixed-8"),
    50:  ("ESP", "variable"),
    51:  ("AH", "variable"),
    59:  ("No Next Header", "none"),
    60:  ("Destination Options", "variable"),
    135: ("Mobility Header", "variable"),
}

UPPER_LAYER = {6: "TCP", 17: "UDP", 58: "ICMPv6", 4: "IPv4", 41: "IPv6"}

def parse_extension_headers(packet: bytes, start_offset: int = 40) -> list:
    """
    Walk the extension header chain starting from start_offset.

    Args:
        packet:       Raw IPv6 packet bytes
        start_offset: Byte offset where the first extension header begins
                      (40 for the first header after IPv6 base header)

    Returns:
        List of (name, next_header, offset, length) tuples
    """
    next_header = packet[6]  # From IPv6 base header
    offset = start_offset
    headers = []

    while offset < len(packet):
        name = EXT_HEADERS.get(next_header, {})
        if next_header in UPPER_LAYER:
            headers.append((UPPER_LAYER[next_header], next_header, offset, -1))
            break
        elif next_header == 59:  # No Next Header
            break
        elif next_header == 44:  # Fragment Header: fixed 8 bytes
            nh = packet[offset]
            headers.append(("Fragment Header", nh, offset, 8))
            next_header = nh
            offset += 8
        elif next_header in EXT_HEADERS:
            nh = packet[offset]
            ext_len = packet[offset + 1]
            length = (ext_len + 1) * 8
            headers.append((EXT_HEADERS[next_header][0], nh, offset, length))
            next_header = nh
            offset += length
        else:
            break

    return headers
```

## Hop-by-Hop vs All Others

The most critical distinction:

```
Hop-by-Hop Options (Next Header = 0):
  ✗ MUST be processed by EVERY router along the path
  ✗ MUST be the FIRST extension header if present
  ✗ Causes performance issues (typically slow-pathed in hardware)
  → Used for: Router Alert, Jumbo Payload
  → In practice: Very rare in production networks

All other extension headers:
  ✓ Processed ONLY by the destination (or specific nodes in routing headers)
  ✓ Transit routers simply forward without examining them
  ✓ No performance impact on forwarding path
  → Used for: fragmentation (44), IPsec (50,51), mobility (135)
```

## Viewing Extension Headers in Practice

```bash
# Capture packets with Hop-by-Hop header
sudo tcpdump -i eth0 -XX "ip6[6] == 0"

# Capture packets with Fragment header
sudo tcpdump -i eth0 -vv "ip6[6] == 44"

# Capture IPsec AH packets
sudo tcpdump -i eth0 "ip6[6] == 51"

# Capture IPsec ESP packets
sudo tcpdump -i eth0 "ip6[6] == 50"
```

## Extension Header Security Considerations

```bash
# Many networks drop packets with unusual extension headers
# RFC 7045 defines rules for which extension headers should be forwarded

# Check if your firewall passes common extension headers:
# Fragment header (44) must be allowed for legitimate fragmented traffic
# AH (51) and ESP (50) must be allowed for IPsec

# ip6tables: allow IPsec headers
sudo ip6tables -A INPUT -p ah -j ACCEPT
sudo ip6tables -A INPUT -p esp -j ACCEPT

# Allow fragmented packets
sudo ip6tables -A INPUT -m frag --fragmore -j ACCEPT  # More fragments
sudo ip6tables -A INPUT -m frag --fraglast -j ACCEPT  # Last fragment
```

## Conclusion

IPv6 extension headers provide a flexible mechanism for optional features that would otherwise require a larger, more complex base header. The key insight is that only Hop-by-Hop Options require per-hop processing — all other extension headers are processed only at the destination. This means transit routers are not burdened by extension header processing in the common case. However, extension headers do present security challenges, and many network operators filter them at boundaries.
