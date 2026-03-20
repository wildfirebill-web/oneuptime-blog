# How to Understand the ORCHIDv2 Address Space (2001:20::/28)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, ORCHIDv2, 2001:20::/28, RFC 7343, Cryptographic, HIP

Description: Understand the ORCHIDv2 address space 2001:20::/28 (RFC 7343), its use for Host Identity Protocol cryptographic identifiers, and why it appears in some environments.

## Introduction

`2001:20::/28` is the ORCHIDv2 (Overlay Routable Cryptographic Hash Identifiers version 2) address space defined in RFC 7343. These addresses are derived from cryptographic hashes and used by the Host Identity Protocol (HIP). They are not routable on the public internet and serve as stable identifiers independent of network topology.

## Key Properties

| Property | Value |
|---|---|
| Prefix | 2001:20::/28 |
| RFC | RFC 7343 (updates RFC 4843) |
| Source | False (not for regular source) |
| Destination | False (not for regular routing) |
| Forwardable | No |
| Globally Reachable | No |

## What is HIP?

Host Identity Protocol (HIP) separates the host identity from its network location:
- **Host Identity (HI)**: A cryptographic public key that uniquely identifies a host
- **ORCHID**: A 128-bit hash derived from the HI, used as an IPv6 address
- **Locator**: The regular IPv6 address used for actual routing

```text
Traditional IPv6:
  address = identity + location  (both in one address)

HIP with ORCHIDv2:
  ORCHID = identity (cryptographic hash, 2001:20::/28)
  IPv6 address = location (changes when host moves)
```

## ORCHID Address Generation

```python
import hashlib
import ipaddress
import struct

ORCHID_PREFIX = 0x2001  # First 16 bits
ORCHID_PREFIX_BITS = 28  # /28

def generate_orchid(context_id: bytes, public_key: bytes) -> str:
    """
    Generate an ORCHIDv2 address (RFC 7343).
    context_id: 128-bit context identifier (defined per application)
    public_key: Host Identity public key bytes
    """
    # Hash input = Context_ID || HI
    hash_input = context_id + public_key

    # SHA1 hash (RFC 4843 used SHA1; ORCHIDv2 uses SHA1 for compatibility)
    sha1 = hashlib.sha1(hash_input).digest()

    # Take the last 96 bits (12 bytes) of the hash
    hash_96 = sha1[-12:]

    # Construct ORCHID: 2001:20::/28 | hash_bits
    # First 28 bits = 0x20012000 >> 4  (we use the first 4 bytes = 32 bits)
    # Actually: first 28 bits are prefix, then 96 bits of hash, total 124...
    # RFC 7343 §3: 28-bit prefix + 4 unused bits + 96 bits hash = 128 bits

    prefix_int = 0x20012000  # 2001:2000::/28 base
    # Shift hash to lower 96 bits
    hash_int = int.from_bytes(sha1[-12:], 'big')
    orchid_int = (prefix_int << 96) | hash_int

    return str(ipaddress.IPv6Address(orchid_int))

# HIP context ID for ESP (RFC 7343 §9.3)

HIP_ESP_CONTEXT = bytes.fromhex("8b9dfb2e9b2a8d8e1d25d5b1a7ba56d6")

# Simulate a host public key
fake_public_key = b"example-host-public-key-material-32b"

orchid = generate_orchid(HIP_ESP_CONTEXT, fake_public_key)
print(f"Generated ORCHID: {orchid}")
print(f"In 2001:20::/28: {ipaddress.IPv6Address(orchid) in ipaddress.IPv6Network('2001:20::/28')}")
```

## Detecting ORCHIDv2 Addresses

```python
import ipaddress

ORCHID_BLOCK = ipaddress.IPv6Network("2001:20::/28")

def is_orchid(addr_str: str) -> bool:
    """Return True if address is an ORCHIDv2 address."""
    try:
        return ipaddress.IPv6Address(addr_str) in ORCHID_BLOCK
    except ValueError:
        return False

# Tests
print(is_orchid("2001:20::1"))          # True
print(is_orchid("2001:2f::1"))          # True (still in /28)
print(is_orchid("2001:30::1"))          # False (outside /28)
print(is_orchid("2001:2::/48"))         # False (benchmarking)
```

## Firewall Filtering

```bash
# ORCHID addresses should not appear in routing
# Block them at network boundaries
ip6tables -A FORWARD -s 2001:20::/28 -j DROP
ip6tables -A FORWARD -d 2001:20::/28 -j DROP
ip6tables -A INPUT -s 2001:20::/28 -j DROP
```

## Conclusion

ORCHIDv2 addresses (`2001:20::/28`) are cryptographic host identifiers used by the Host Identity Protocol. They are not routable and should be filtered at network boundaries. If you see ORCHID addresses in traffic logs, it may indicate HIP-capable applications or misconfiguration. Monitor for unexpected ORCHID traffic with OneUptime security monitoring.
