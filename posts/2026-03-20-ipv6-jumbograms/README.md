# How to Understand IPv6 Jumbograms

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Jumbograms, Jumbo Payload, RFC 2675, HPC Networking

Description: Understand IPv6 jumbograms — packets exceeding 65535 bytes — how they work using the Jumbo Payload option, when they are useful, and their requirements.

## Introduction

IPv6's Payload Length field is 16 bits, supporting a maximum payload of 65535 bytes. For high-performance computing, storage area networks, and InfiniBand over Ethernet, larger packets are desirable to reduce per-packet overhead. IPv6 jumbograms (RFC 2675) extend the maximum payload beyond 65535 bytes using a Hop-by-Hop option, enabling payloads up to 4,294,967,295 bytes (approximately 4 GB).

## Standard IPv6 vs Jumbogram Size Limits

```
Standard IPv6 packet:
  Payload Length field: 16 bits → max 65535 bytes payload
  Total packet size: 40 (header) + 65535 = 65575 bytes maximum

IPv6 Jumbogram (RFC 2675):
  Payload Length field: set to 0 (signals jumbogram)
  Jumbo Payload Length: 32-bit field in Hop-by-Hop option
  Maximum payload: 2^32 - 1 = 4,294,967,295 bytes
  Total packet size: up to ~4 GB

Practical maximum (limited by link MTU):
  Standard Ethernet: 1500 bytes (no jumbogram benefit)
  Jumbo frames (9000 bytes): modest benefit
  InfiniBand: up to 65535-byte MTU
  HPC interconnects: can support larger MTUs for jumbograms
```

## The Jumbo Payload Hop-by-Hop Option

Jumbograms use a Hop-by-Hop Options extension header with the Jumbo Payload option:

```
Hop-by-Hop Options Header with Jumbo Payload option:

Byte 0: Next Header (type of header after Hop-by-Hop)
Byte 1: Hdr Ext Len = 0 (8-byte header total)
Byte 2: Option Type = 0xC2 (Jumbo Payload, 2 bits: 11 = skip+discard)
Byte 3: Option Length = 4 (the value field is 4 bytes)
Byte 4-7: Jumbo Payload Length (32-bit big-endian)

When a jumbogram is sent:
  1. IPv6 Payload Length field is set to 0
  2. Hop-by-Hop Options header is added as the first extension header
  3. Hop-by-Hop Options contains the Jumbo Payload option
  4. Jumbo Payload Length contains the actual total payload size (≥ 65536)
```

## Jumbogram Requirements and Constraints

```
RFC 2675 requirements:

1. Must not be fragmented
   → Jumbograms cannot use the Fragment Header
   → The source must ensure the entire jumbogram fits in the path MTU

2. Hop-by-Hop Options header must be first extension header
   → All routers process Hop-by-Hop options
   → Jumbo Payload Length reaches every router

3. Routers must support jumbograms to forward them
   → A router that cannot handle jumbograms sends ICMPv6 Parameter Problem
   → Code 0, Pointer to the Jumbo Payload option

4. Link must support the required MTU
   → Useless on standard Ethernet (1500-byte MTU)
   → Designed for high-speed interconnects with large MTUs

5. Upper-layer protocols must handle large sizes
   → TCP: uses 32-bit sequence numbers (handles jumbogram sizes)
   → UDP: checksum remains 16-bit, payload up to 4 GB
```

## Building a Jumbogram Hop-by-Hop Header

```python
import struct

def build_jumbo_payload_header(next_header: int,
                                jumbo_length: int) -> bytes:
    """
    Build an IPv6 Hop-by-Hop Options header with Jumbo Payload option.
    Used to create IPv6 jumbograms (RFC 2675).

    Args:
        next_header:  Type of the following header (6=TCP, 17=UDP)
        jumbo_length: Total payload size (must be >= 65536)

    Returns:
        8-byte Hop-by-Hop Options header with Jumbo Payload option
    """
    if jumbo_length < 65536:
        raise ValueError(f"Jumbo payload length must be >= 65536, got {jumbo_length}")
    if jumbo_length > 0xFFFFFFFF:
        raise ValueError("Jumbo payload length exceeds 32-bit maximum")

    JUMBO_PAYLOAD_OPTION_TYPE = 0xC2

    return struct.pack(
        "!BBBBI",
        next_header,              # Next Header
        0,                        # Hdr Ext Len = 0 (8 bytes total)
        JUMBO_PAYLOAD_OPTION_TYPE,# Jumbo Payload option type
        4,                        # Option data length = 4 bytes
        jumbo_length              # 32-bit jumbo payload length
    )

def parse_jumbo_payload_header(data: bytes) -> dict:
    """Parse a Hop-by-Hop Options header for the Jumbo Payload option."""
    if len(data) < 8:
        raise ValueError("Need at least 8 bytes")

    next_header, hdr_ext_len = data[0], data[1]
    total_bytes = (hdr_ext_len + 1) * 8

    # Scan for Jumbo Payload option (type 0xC2)
    offset = 2
    while offset < total_bytes:
        opt_type = data[offset]
        if opt_type == 0:     # Pad1
            offset += 1
            continue
        opt_len = data[offset + 1]
        if opt_type == 0xC2:  # Jumbo Payload
            jumbo_length = struct.unpack("!I", data[offset+2:offset+6])[0]
            return {"has_jumbo": True, "jumbo_length": jumbo_length, "next_header": next_header}
        offset += 2 + opt_len

    return {"has_jumbo": False, "next_header": next_header}

# Example: Build header for a 100,000-byte jumbogram
header = build_jumbo_payload_header(17, 100000)  # 17 = UDP
print(f"Hop-by-Hop header: {header.hex()}")

parsed = parse_jumbo_payload_header(header)
print(f"Jumbo length: {parsed['jumbo_length']} bytes")
```

## Conclusion

IPv6 jumbograms address the 65535-byte payload limit by using a 32-bit Jumbo Payload Length option in a Hop-by-Hop Options header. They are applicable only on high-performance network links that support MTUs above 65535 bytes — InfiniBand, some HPC interconnects, and specialized storage networks. On standard Ethernet (1500 or 9000 bytes jumbo frames), jumbograms provide no benefit. The most important constraint: jumbograms cannot be fragmented, so the link MTU must be large enough to carry the entire packet.
