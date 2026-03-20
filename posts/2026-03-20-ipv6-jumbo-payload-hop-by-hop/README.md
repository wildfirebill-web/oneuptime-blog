# How to Understand the IPv6 Jumbo Payload Hop-by-Hop Option

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Hop-by-Hop Options, Jumbo Payload, Jumbograms, Extension Headers

Description: Understand how the Jumbo Payload Hop-by-Hop option enables IPv6 packets larger than 65535 bytes, its encoding rules, and why it must be a Hop-by-Hop option.

## Introduction

The Jumbo Payload option (RFC 2675) is a Hop-by-Hop option that enables IPv6 jumbograms — packets with payloads larger than 65535 bytes. It must be placed in a Hop-by-Hop Options header because every router on the path needs to read and process the actual payload length. The option type value (0xC2) is carefully chosen so that routers that don't understand it will skip it and keep forwarding.

## Why It Must Be a Hop-by-Hop Option

```
Options in IPv6 headers that must be processed by every router:
  → Must use the Hop-by-Hop Options header
  → Hop-by-Hop header is the FIRST extension header
  → Every router reads and processes Hop-by-Hop options

Contrast with Destination Options:
  → Processed only by the final destination
  → Cannot be used for Jumbo Payload (routers need the length)

Why routers need the Jumbo Payload length:
  1. Forwarding decisions based on packet completeness
  2. Rate accounting and QoS based on actual packet size
  3. Proper handling of ICMPv6 errors referencing the packet
  4. Validating that the packet arrives complete
```

## Option Type Encoding: 0xC2

The option type byte encodes behavior for unknown options:

```
Option Type 0xC2 = 11000010 binary

Bits 7-6 (top 2): 11 = "skip over this option and continue processing"
  Other values:
    00 = skip quietly
    01 = discard packet quietly
    10 = discard and send ICMPv6 Parameter Problem (not multicast dest)
    11 = discard and send ICMPv6 Parameter Problem (any dest)

Wait — 0xC2 = 11000010:
  Bits 7-6: 11 = discard + send ICMPv6 to any destination
  Bit 5:    0  = option data does NOT change en route
  Bits 4-0: 00010 = option type identifier

Actually RFC 2675 uses 0xC2 which means if unknown:
  discard the packet and send ICMPv6 Parameter Problem
  This ensures old routers reject jumbograms loudly rather than
  silently forwarding an incorrectly-sized packet
```

## Complete Option Encoding

```python
import struct

# Jumbo Payload option encoding (RFC 2675)
JUMBO_PAYLOAD_OPTION_TYPE = 0xC2
JUMBO_PAYLOAD_OPTION_LENGTH = 4   # 4 bytes of option data

def encode_jumbo_payload_option(jumbo_payload_length: int) -> bytes:
    """
    Encode the Jumbo Payload TLV option.
    Returns just the option bytes (to be placed inside Hop-by-Hop header).
    """
    if jumbo_payload_length < 65536:
        raise ValueError("Jumbo payload length must be at least 65536")

    return struct.pack(
        "!BBI",
        JUMBO_PAYLOAD_OPTION_TYPE,   # Type: 0xC2
        JUMBO_PAYLOAD_OPTION_LENGTH, # Length: 4 bytes
        jumbo_payload_length         # 32-bit length value
    )

def build_hop_by_hop_with_jumbo(next_header: int,
                                  jumbo_length: int) -> bytes:
    """
    Build a complete Hop-by-Hop Options header containing the
    Jumbo Payload option, padded to a multiple of 8 bytes.

    Args:
        next_header:  Next Header type following Hop-by-Hop
        jumbo_length: Actual payload length (>= 65536)

    Returns:
        Complete Hop-by-Hop Options header (8 bytes minimum)
    """
    option = encode_jumbo_payload_option(jumbo_length)  # 6 bytes

    # Hop-by-Hop header must be multiple of 8 bytes
    # Fixed 2 bytes (next_header + hdr_ext_len) + 6 bytes option = 8 bytes total
    # Hdr Ext Len = (total_bytes / 8) - 1 = (8/8) - 1 = 0

    return struct.pack(
        "!BB",
        next_header,  # Next Header
        0,            # Hdr Ext Len = 0 (8 bytes total)
    ) + option

def decode_hop_by_hop_jumbo(data: bytes) -> dict:
    """
    Decode a Hop-by-Hop Options header, extracting Jumbo Payload if present.
    """
    if len(data) < 2:
        return {"error": "Too short"}

    next_header = data[0]
    hdr_ext_len = data[1]
    total_bytes = (hdr_ext_len + 1) * 8

    result = {
        "next_header": next_header,
        "total_bytes": total_bytes,
        "jumbo_payload_length": None,
    }

    offset = 2
    while offset < total_bytes and offset < len(data):
        opt_type = data[offset]
        if opt_type == 0:          # Pad1: single byte, no length
            offset += 1
            continue
        if offset + 1 >= len(data):
            break
        opt_len = data[offset + 1]
        if opt_type == JUMBO_PAYLOAD_OPTION_TYPE and opt_len == 4:
            jp_len = struct.unpack("!I", data[offset+2:offset+6])[0]
            result["jumbo_payload_length"] = jp_len
        offset += 2 + opt_len

    return result

# Test encode/decode round-trip
payload_size = 100_000  # 100 KB jumbogram
header = build_hop_by_hop_with_jumbo(17, payload_size)  # 17 = UDP
print(f"Hop-by-Hop header: {header.hex()} ({len(header)} bytes)")

decoded = decode_hop_by_hop_jumbo(header)
print(f"Decoded: next_header={decoded['next_header']}, jumbo_length={decoded['jumbo_payload_length']}")
```

## Rules When Using the Jumbo Payload Option

```
RFC 2675 rules for jumbogram senders:

1. IPv6 Payload Length field MUST be set to 0
   → Signals to the receiver that a Jumbo Payload option is present

2. The Hop-by-Hop Options header MUST be the first extension header
   → Placed immediately after the IPv6 base header

3. The Jumbo Payload option MUST be the first option in the Hop-by-Hop header
   (or at least present; order within Hop-by-Hop is flexible)

4. The packet MUST NOT contain a Fragment Header
   → Jumbograms cannot be fragmented
   → Violating this is invalid; receiver discards

5. Upper-layer length fields must use the Jumbo Payload Length
   → TCP: no change (32-bit sequence space handles it)
   → UDP: length field set to 0 (per RFC 2675 Section 4)
   → ICMPv6: uses Jumbo Payload Length in pseudo-header checksum
```

## Conclusion

The Jumbo Payload Hop-by-Hop option is a carefully designed mechanism that extends IPv6 beyond its 65535-byte payload limit. Its placement in the Hop-by-Hop Options header ensures every router processes it. The option type value 0xC2 ensures old routers that don't understand jumbograms reject them explicitly with an ICMPv6 error rather than silently mishandling them. In practice, jumbograms are rare and only applicable on specialized high-performance networks where every link supports the required large MTU.
