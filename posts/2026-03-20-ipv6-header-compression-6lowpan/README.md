# How to Understand IPv6 Header Compression in 6LoWPAN

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, 6LoWPAN, Header Compression, IPHC, IoT, Networking

Description: Understand how 6LoWPAN's IPHC (IP Header Compression) scheme compresses 40-byte IPv6 headers to a few bytes, making IPv6 practical for IEEE 802.15.4 constrained networks.

## Introduction

The IPv6 fixed header is 40 bytes. On IEEE 802.15.4 with a maximum payload of ~100 bytes, this leaves only 60 bytes for useful data after accounting for headers. 6LoWPAN's IPHC (IP Header Compression) scheme, defined in RFC 6282, reduces the IPv6 header to as few as 2 bytes in the best case.

## The IPv6 Header Structure

The standard IPv6 header contains:

| Field | Bits | Bytes | Notes |
|---|---|---|---|
| Version | 4 | 0.5 | Always 6 |
| Traffic Class | 8 | 1 | Usually 0 |
| Flow Label | 20 | 2.5 | Usually 0 in IoT |
| Payload Length | 16 | 2 | Can be inferred |
| Next Header | 8 | 1 | Usually UDP (17) |
| Hop Limit | 8 | 1 | Usually 64 |
| Source Address | 128 | 16 | Can be compressed |
| Destination Address | 128 | 16 | Can be compressed |
| **Total** | | **40** | |

## IPHC Compression Techniques

### 1. Traffic Class and Flow Label Elision

In most IoT traffic, Traffic Class = 0 and Flow Label = 0. IPHC can elide (completely omit) both fields:

```
Original: Version(4b) + TC(8b) + Flow Label(20b) = 32 bits = 4 bytes
IPHC:     If TC=0 and FL=0, both are elided → 0 bytes
Savings:  4 bytes
```

### 2. Next Header Compression

Common next headers (UDP=17, TCP=6, ICMPv6=58) can be compressed using a single bit indicating "use NHC" (Next Header Compression):

```
Original: Next Header field = 1 byte (e.g., 0x11 for UDP)
IPHC:     "TF" bit + NHC header = 1 byte total for common protocols
```

### 3. Hop Limit

Hop limit can be compressed to one of three common values using a 2-bit field:

```
00: Hop limit is inline (not compressed)
01: Hop limit = 1 (elided)
10: Hop limit = 64 (elided, most common)
11: Hop limit = 255 (elided)
Savings: 1 byte for common values
```

### 4. Address Compression

Address compression is the biggest win. IPHC uses context information (the shared prefix) to elide address bytes:

```
Stateless compression modes:
- "Fully elided" (SAC=0, SAM=11): Source = link-local from IID (0 bytes inline)
- "16-bit inline" (SAC=0, SAM=10): Source = prefix + 16-bit short address (2 bytes)
- "64-bit inline" (SAC=0, SAM=01): Source = prefix + 64-bit IID (8 bytes)
- "128-bit inline" (SAC=0, SAM=00): Full 16-byte address (16 bytes)

Stateful compression (with context):
- SAC=1, SAM=11: Address fully elided (0 bytes) using stored context prefix
```

## Practical Compression Example

A typical sensor-to-gateway UDP packet:

```
Uncompressed IPv6+UDP headers: 40 + 8 = 48 bytes

With IPHC compression:
- IPHC dispatch byte: 2 bytes
- Version/TC/FL: elided (all default values)
- Payload length: elided (inferred from frame size)
- Next header: compressed (NHC)
- Hop limit: 1 byte elided (64)
- Source address: 0 bytes (link-local, fully derived from MAC)
- Destination address: 2 bytes (short address form)
- NHC UDP: ports elided (both well-known) = 1 byte
- UDP length: elided
- UDP checksum: 2 bytes or elided

Result: ~4-7 bytes for IPv6+UDP headers
Savings: 41-44 bytes (87-92% compression)
```

## 6LoWPAN Packet Capture Analysis

```bash
# Capture 6LoWPAN frames on a Linux 802.15.4 interface
sudo tcpdump -i lowpan0 -v

# Alternatively, use Wireshark with 6LoWPAN dissector
# Filter: wpan or 6lowpan

# On a Contiki-NG node, enable 6LoWPAN debug logging:
# project-conf.h:
# #define SICSLOWPAN_CONF_COMPRESSION_THRESHOLD 0
# #define LOG_CONF_LEVEL_6LOWPAN LOG_LEVEL_DBG
```

## Compression Context

Stateful IPHC uses "contexts" — shared prefix knowledge — for maximum compression:

```
Context 0: fe80::/64 (always available for link-local)
Context 1: prefix assigned by border router (e.g., 2001:db8:mesh:1::/64)

With context 1, a global unicast address 2001:db8:mesh:1::1234:5678:9abc:def0
can be compressed to just the 8-byte IID: 1234:5678:9abc:def0
or further to 2 bytes if a short address is used.
```

## Conclusion

6LoWPAN IPHC compression transforms the 40-byte IPv6 header into 2-7 bytes for typical IoT traffic patterns, making IPv6 practical on IEEE 802.15.4 links with 80-100 byte payloads. The compression works by exploiting the fact that many IPv6 header fields are constant or predictable in IoT traffic (version, traffic class, flow label, hop limit) and using context tables to elide address prefixes. Understanding IPHC is essential for troubleshooting and optimizing IPv6 IoT network performance.
