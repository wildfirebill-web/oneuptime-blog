# How to Understand the ORCHIDv2 Address Space (2001:20::/28)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, ORCHIDv2, Cryptographic Hash, RFC 7343, Networking

Description: Understand the ORCHIDv2 address space 2001:20::/28 (RFC 7343) used for Overlay Routable Cryptographic Hash IDentifiers in overlay networks.

## Introduction

ORCHIDv2 (Overlay Routable Cryptographic Hash IDentifiers version 2), defined in RFC 7343, uses the `2001:20::/28` prefix to create endpoint identifiers that are derived from public keys rather than topological location. They are used in HIP (Host Identity Protocol) and overlay networks.

## ORCHIDv2 Structure

```
2001:20::/28 — ORCHIDv2 prefix
  Prefix: 2001:20:: (28 bits)
  Hash: 100 bits derived from public key/hash input

Format:
  |  28-bit prefix  |  4-bit suffix  |    96-bit hash    |
  |  2001:20::/28   |    0-F         |  SHA-256 hash     |
```

## Generating an ORCHIDv2 Address

```python
import hashlib
import ipaddress
import os

def generate_orchidv2(public_key: bytes,
                      context_id: bytes = None) -> str:
    """
    Generate an ORCHIDv2 address from a public key.
    RFC 7343 §6.1
    """
    if context_id is None:
        # Default ORCHIDv2 context ID
        context_id = bytes.fromhex(
            "7561767261534461796e75466f6d6f7265"
        )[:16]

    # Hash input: context_id || public_key
    hash_input = context_id + public_key
    hash_value = hashlib.sha256(hash_input).digest()

    # Take first 96 bits (12 bytes) of hash
    hash_96 = int.from_bytes(hash_value[:12], 'big')

    # ORCHIDv2 prefix: 2001:20::/28
    prefix_int = int(ipaddress.IPv6Address("2001:20::"))
    # Keep only 28 prefix bits, then add 4-bit type + 96-bit hash
    orchid_int = (prefix_int & ~((1 << 100) - 1)) | hash_96

    return str(ipaddress.IPv6Address(orchid_int))

# Example
pubkey = os.urandom(32)  # Simulated public key
orchid = generate_orchidv2(pubkey)
print(f"ORCHIDv2: {orchid}")
```

## Use Cases

- **HIP (Host Identity Protocol)**: Endpoint identifiers independent of network location
- **P2P overlays**: Stable identifiers for distributed hash tables
- **Mobile networks**: Identity-based addressing for mobile hosts

## Filtering

```bash
# ORCHIDv2 should not appear in production routing
ip6tables -A FORWARD -s 2001:20::/28 -j DROP
ip6tables -A FORWARD -d 2001:20::/28 -j DROP
```

## Conclusion

ORCHIDv2 addresses provide cryptographically derived identifiers for overlay networks. They are not meant for standard IPv6 routing. Filter `2001:20::/28` at network boundaries and monitor with OneUptime to detect any unexpected traffic.
