# How to Understand the Authentication Header (AH) in IPv6 - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, IPsec, AH, Authentication, Security

Description: Learn how the IPv6 Authentication Header provides data integrity and authentication without encryption, and understand its structure, limitations with NAT, and practical use cases.

## Overview

The Authentication Header (AH) is IPsec protocol 51 (Next Header value 51 in IPv6). It provides data integrity and authentication for IPv6 packets - ensuring the packet came from a legitimate sender and was not modified in transit. AH does NOT provide confidentiality (no encryption). It is rarely used alone today, as ESP with authentication provides both integrity and encryption.

## AH Header Structure

```text
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| Next Header   |  Payload Len  |          RESERVED             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                 Security Parameters Index (SPI)               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Sequence Number Field                      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                Integrity Check Value (ICV)                    +
|             (variable length, typically 12 or 16 bytes)       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

- **SPI**: 32-bit identifier - with destination IP and protocol, uniquely identifies the SA
- **Sequence Number**: Anti-replay protection (increments with each packet)
- **ICV**: HMAC-SHA1-96 or HMAC-SHA256-128 over the packet

## What AH Authenticates

AH calculates an ICV over the IPv6 header, extension headers, and payload, but mutable fields are zeroed:

**Covered by AH (immutable):**
- IPv6 version, payload length
- Next Header field
- Source and Destination addresses
- All extension headers (with mutable fields zeroed)
- TCP/UDP header and payload

**Zeroed for AH calculation (mutable):**
- Traffic Class (DSCP/ECN may be rewritten)
- Flow Label
- Hop Limit (decremented at each hop)

## AH in Transport Mode (Host-to-Host)

In transport mode, AH is inserted between the IPv6 header and the upper-layer header:

```text
Before AH:
[IPv6: src=A, dst=B, NH=TCP] [TCP] [Data]

After AH Transport Mode:
[IPv6: src=A, dst=B, NH=51] [AH: Next=TCP, SPI=0x1234, ICV=...] [TCP] [Data]
```

## AH in Tunnel Mode (Gateway-to-Gateway)

In tunnel mode, the original packet is encapsulated:

```text
[Outer IPv6: src=GW1, dst=GW2, NH=51] [AH: Next=IPv6, ICV=...] [Inner IPv6] [TCP] [Data]
```

## Configuring AH on Linux with ip xfrm

```bash
# Create AH Security Association (SA)

ip xfrm state add \
  src 2001:db8:1::1 dst 2001:db8:2::1 \
  proto ah spi 0x100 \
  auth hmac\(sha256\) 0x0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef \
  mode transport

ip xfrm state add \
  src 2001:db8:2::1 dst 2001:db8:1::1 \
  proto ah spi 0x200 \
  auth hmac\(sha256\) 0xfedcba9876543210fedcba9876543210fedcba9876543210fedcba9876543210 \
  mode transport

# Create Security Policy (SP)
ip xfrm policy add \
  src 2001:db8:1::1/128 dst 2001:db8:2::1/128 \
  proto any dir out \
  tmpl src 2001:db8:1::1 dst 2001:db8:2::1 proto ah mode transport

ip xfrm policy add \
  src 2001:db8:2::1/128 dst 2001:db8:1::1/128 \
  proto any dir in \
  tmpl src 2001:db8:2::1 dst 2001:db8:1::1 proto ah mode transport

# Verify
ip xfrm state list
ip xfrm policy list
```

## Verify AH Traffic with tcpdump

```bash
# Capture AH traffic (protocol 51)
tcpdump -i eth0 'ip6 proto 51' -n -v

# Output example:
# 2001:db8:1::1 > 2001:db8:2::1: AH(spi=0x00000100,seq=0x1)
#   length=20  auth hmac-sha256
```

## Why AH Is Rarely Used Alone

1. **No encryption**: Data is visible in plaintext - any eavesdropper can read it
2. **NAT incompatibility**: AH authenticates the IP header including addresses - NAT changes the source address, breaking AH verification
3. **ESP does both**: ESP with authentication provides integrity + confidentiality - superset of AH
4. **Deployment complexity**: Most IPsec deployments use ESP-only or ESP+AH in high-security scenarios

## When AH Makes Sense

AH is appropriate when:
- Confidentiality is not required (performance optimization - no encryption)
- Routing infrastructure needs to authenticate packets without decrypting them
- Protocol-level integrity verification is needed in addition to application-layer TLS
- Combined with ESP in very high-security deployments (AH outside ESP)

## AH + ESP (Maximum Security)

For maximum protection:

```text
[IPv6] [AH] [ESP] [Encrypted TCP/Data] [ESP-Trailer] [ESP-Auth]

AH authenticates: IPv6 header + AH header + ESP header + encrypted payload
ESP authenticates: payload only (inside encryption)
Combined: both outer and inner integrity verified
```

```bash
# Rarely needed, but possible with ip xfrm by stacking SA entries
ip xfrm policy add src ... dst ... tmpl proto ah ... tmpl proto esp ...
```

## Summary

AH (Authentication Header, NH=51) provides integrity and authentication for IPv6 packets without encryption. It inserts between the IPv6 header and upper-layer protocol, calculating an HMAC over immutable header fields and payload. AH is incompatible with NAT (because NAT modifies the source address, invalidating the ICV) and is rarely used alone - most deployments use ESP with authentication (which provides both integrity and confidentiality). AH is still relevant in specific high-security scenarios requiring outer header integrity verification.
