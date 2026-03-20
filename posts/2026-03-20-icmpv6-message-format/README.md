# How to Understand ICMPv6 Message Format

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ICMPv6, IPv6, Message Format, RFC 4443, Networking

Description: Understand the structure of ICMPv6 messages, the common header fields, how Type and Code distinguish different messages, and the checksum covering the IPv6 pseudo-header.

## Introduction

ICMPv6 (Internet Control Message Protocol for IPv6) is defined in RFC 4443 and serves as the diagnostic and control protocol for IPv6, replacing both ICMPv4 and ARP. ICMPv6 carries error messages, informational messages, Neighbor Discovery, Router Discovery, and multicast group management. All ICMPv6 messages share a common 4-byte header, with the remaining structure specific to each message type.

## ICMPv6 Common Header Structure

```
ICMPv6 message structure:

 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|     Type      |     Code      |          Checksum             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                Message Body (type-specific)                   +
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

Fields:
  Type [8 bits]:     Identifies the ICMPv6 message category
  Code [8 bits]:     Sub-type within the message category
  Checksum [16 bits]: Covers ICMPv6 header + body + IPv6 pseudo-header
  Body:              Varies by Type/Code
```

## ICMPv6 Type Registry

```
Error Messages (Types 1-127):
  Type 1:   Destination Unreachable
  Type 2:   Packet Too Big
  Type 3:   Time Exceeded
  Type 4:   Parameter Problem
  Type 100: Private experimentation
  Type 101: Private experimentation

Informational Messages (Types 128-255):
  Type 128: Echo Request (ping6)
  Type 129: Echo Reply
  Type 130: Multicast Listener Query (MLDv1)
  Type 131: Multicast Listener Report (MLDv1)
  Type 132: Multicast Listener Done (MLDv1)
  Type 133: Router Solicitation (NDP)
  Type 134: Router Advertisement (NDP)
  Type 135: Neighbor Solicitation (NDP)
  Type 136: Neighbor Advertisement (NDP)
  Type 137: Redirect
  Type 143: MLDv2 Report
```

## Parsing ICMPv6 Messages

```python
import struct
import socket

ICMPV6_TYPES = {
    1:   "Destination Unreachable",
    2:   "Packet Too Big",
    3:   "Time Exceeded",
    4:   "Parameter Problem",
    128: "Echo Request",
    129: "Echo Reply",
    133: "Router Solicitation",
    134: "Router Advertisement",
    135: "Neighbor Solicitation",
    136: "Neighbor Advertisement",
    137: "Redirect",
    143: "MLDv2 Report",
}

ICMPV6_CODES = {
    1: {0: "No route to destination", 1: "Communication prohibited",
        2: "Beyond scope of source", 3: "Address unreachable",
        4: "Port unreachable", 5: "Failed ingress/egress policy",
        6: "Reject route", 7: "Error in Source Routing Header"},
    2: {0: "Packet Too Big"},
    3: {0: "Hop limit exceeded in transit", 1: "Fragment reassembly time exceeded"},
    4: {0: "Erroneous header field", 1: "Unrecognized Next Header type",
        2: "Unrecognized IPv6 option"},
}

def parse_icmpv6_header(data: bytes) -> dict:
    """Parse the common ICMPv6 4-byte header."""
    if len(data) < 4:
        raise ValueError("ICMPv6 header requires at least 4 bytes")

    icmp_type, code, checksum = struct.unpack("!BBH", data[:4])

    type_name = ICMPV6_TYPES.get(icmp_type, f"Unknown ({icmp_type})")
    code_name = ICMPV6_CODES.get(icmp_type, {}).get(code, f"Code {code}")

    return {
        "type": icmp_type,
        "type_name": type_name,
        "code": code,
        "code_name": code_name,
        "checksum": checksum,
        "is_error": icmp_type < 128,
        "is_informational": icmp_type >= 128,
        "body": data[4:],
    }

# Parse each error type
test_messages = [
    (b'\x01\x04\x00\x00' + b'\x00' * 4, "Destination Unreachable, Port Unreachable"),
    (b'\x02\x00\x00\x00' + struct.pack("!I", 1280), "Packet Too Big, MTU=1280"),
    (b'\x03\x01\x00\x00' + b'\x00' * 4, "Time Exceeded, Reassembly"),
    (b'\x80\x00\x00\x00' + b'\x00' * 4, "Echo Request"),
]

for msg, description in test_messages:
    parsed = parse_icmpv6_header(msg)
    print(f"{description}:")
    print(f"  Type={parsed['type']} ({parsed['type_name']}), Code={parsed['code']} ({parsed['code_name']})")
    print(f"  Is error: {parsed['is_error']}")
```

## ICMPv6 Checksum

Unlike ICMPv4, ICMPv6 checksums are mandatory and cover a pseudo-header including IPv6 addresses:

```
ICMPv6 checksum covers:
  1. IPv6 pseudo-header:
     - Source address (128 bits)
     - Destination address (128 bits)
     - ICMPv6 payload length (32 bits)
     - Next Header = 58 (32 bits, zero-padded)
  2. ICMPv6 header (Type + Code + Checksum=0 + Body)

This binds the ICMPv6 message to its IPv6 source and destination,
preventing spoofed ICMPv6 messages even if IPv6 source address
verification is not available.
```

## Conclusion

ICMPv6 is a fundamental IPv6 protocol that handles error reporting, path discovery, and neighbor/router discovery. Its common 4-byte header (Type, Code, Checksum, Body) provides a consistent structure for all message types. Error messages use Types 1-127, informational messages use Types 128-255. The mandatory checksum covering both the message and the IPv6 pseudo-header provides integrity protection. Understanding the Type/Code taxonomy is essential for writing firewall rules, debugging connectivity, and implementing IPv6 network applications.
