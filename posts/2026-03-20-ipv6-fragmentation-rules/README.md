# How to Understand IPv6 Fragmentation Rules

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Fragmentation, MTU, Path MTU Discovery, RFC 8200

Description: Understand the rules governing IPv6 packet fragmentation, how it differs fundamentally from IPv4, and what the source must do when packets exceed the path MTU.

## Introduction

IPv6 fragmentation follows fundamentally different rules than IPv4. In IPv4, any router can fragment a packet; in IPv6, only the source can fragment. This architectural change improves router performance but places more responsibility on the source. Understanding these rules is essential for implementing IPv6 applications and troubleshooting connectivity issues.

## Core IPv6 Fragmentation Rules (RFC 8200)

```text
Rule 1: Only the source can fragment
  → No intermediate router may fragment an IPv6 packet
  → A router that receives a packet too large for the next link:
    MUST drop the packet
    MUST send ICMPv6 "Packet Too Big" to the source

Rule 2: The source must discover the Path MTU
  → Before sending, source performs Path MTU Discovery
  → Initial assumption: Path MTU = minimum of local interface MTU
  → When ICMPv6 Packet Too Big received: reduce packet size

Rule 3: Minimum MTU = 1280 bytes
  → Every IPv6 link must support at least 1280 bytes
  → Source may use 1280-byte packets without PMTUD on unknown-MTU paths

Rule 4: Fragmentation uses the Fragment Extension Header
  → Fragment header (NH=44) inserted between base header and payload
  → Each fragment contains: IPv6 base + Fragment header + payload chunk
  → Fragment header carries: offset, more-flag, identification

Rule 5: Fragment offset must be multiple of 8
  → Fragment Offset field is in units of 8 bytes
  → All fragments except the last must be a multiple of 8 bytes

Rule 6: Atomic fragments
  → A source may also fragment a packet that fits in the MTU
    (e.g., to avoid fragmentation at a smaller-MTU link later)
  → Not common but valid
```

## IPv6 vs IPv4 Fragmentation Comparison

```text
IPv4:
  Any router can fragment       → simpler for sources, more load on routers
  DF bit controls fragmentability → sources can prevent router fragmentation
  Fragmentation is common       → many routers fragment frequently
  Min router MTU: 68 bytes      → very low practical minimum

IPv6:
  Only source can fragment      → eliminates router fragmentation overhead
  All packets can be fragmented by source → source must track PMTU
  PMTUD required for efficiency → routers must deliver ICMPv6 Packet Too Big
  Min link MTU: 1280 bytes      → higher minimum than IPv4
```

## The Source Fragmentation Process

```python
import struct
import socket
import os

class IPv6Fragmenter:
    """Implements IPv6 source fragmentation per RFC 8200."""

    def __init__(self, path_mtu: int = 1500):
        self.path_mtu = path_mtu
        self._identification = 0

    def _next_id(self) -> int:
        """Generate next unique fragment identification."""
        self._identification = (self._identification + 1) & 0xFFFFFFFF
        return self._identification

    def needs_fragmentation(self, payload_bytes: int,
                             extra_headers_bytes: int = 0) -> bool:
        """Return True if the payload needs fragmentation."""
        ipv6_total = 40 + extra_headers_bytes + payload_bytes
        return ipv6_total > self.path_mtu

    def fragment_payload(self, payload: bytes,
                          src: str, dst: str,
                          next_header: int) -> list:
        """
        Fragment a payload into IPv6 packets.

        Args:
            payload:     The data to fragment (after any pre-fragment headers)
            src:         Source IPv6 address
            dst:         Destination IPv6 address
            next_header: The Next Header value for the original payload

        Returns:
            List of (fragment_offset, data, more_flag) tuples
        """
        IPV6_HEADER = 40
        FRAG_HEADER = 8

        # Maximum data per fragment (must be multiple of 8)
        max_data = (self.path_mtu - IPV6_HEADER - FRAG_HEADER) // 8 * 8

        identification = self._next_id()
        fragments = []
        offset = 0

        while offset < len(payload):
            chunk = payload[offset:offset + max_data]
            more = 1 if (offset + max_data) < len(payload) else 0
            fragments.append({
                "offset": offset,
                "offset_field": offset // 8,  # In 8-byte units
                "data": chunk,
                "more_flag": more,
                "identification": identification,
                "next_header": next_header,
            })
            offset += len(chunk)

        return fragments

# Example: fragment a 3000-byte TCP segment

fragmenter = IPv6Fragmenter(path_mtu=1500)
tcp_plus_data = bytes(3000)  # Simulate TCP header + data

if fragmenter.needs_fragmentation(len(tcp_plus_data)):
    frags = fragmenter.fragment_payload(tcp_plus_data, "2001:db8::1", "2001:db8::2", 6)
    print(f"Fragmented into {len(frags)} pieces:")
    for f in frags:
        print(f"  Offset {f['offset']}: {len(f['data'])} bytes, More={f['more_flag']}")
```

## Fragment Identification

The 32-bit Identification field links all fragments of the same original packet:

```text
Identification rules:
  - Must be unique for the same (source, destination, next-header) triple
    for the duration the fragments could be in-flight
  - RFC 7739 recommends making Identification pseudo-random
  - Not required to be sequential
  - Must remain unique for at least the maximum packet lifetime (~60 seconds)
```

## Atomic Fragments

RFC 6946 addressed a security issue with "atomic fragments" - single packets that have a Fragment Header but are not actually fragmented (offset=0, M=0):

```bash
# Atomic fragments can be forced by:
# 1. A router spoofing ICMPv6 "Packet Too Big" with very large MTU
# 2. Source sending a single fragment for other reasons

# Linux can generate atomic fragments; check settings
cat /proc/sys/net/ipv6/conf/all/use_tempaddr

# RFC 6946: Hosts MUST handle atomic fragments
# Treat single-fragment packets as regular fragments for reassembly
```

## Conclusion

IPv6 fragmentation places the entire responsibility on the source. Routers that receive oversized packets send ICMPv6 Packet Too Big and drop the packet - they never fragment. This requires sources to implement Path MTU Discovery correctly and to create proper Fragment Headers when fragmentation is needed. Fragment offset must be in multiples of 8 bytes, the identification must be unique within a reasonable timeframe, and all fragments except the last must be a multiple of 8 bytes. Understanding these rules is prerequisite to implementing any IPv6 application that sends large data.
