# How to Check If an IPv6 Address Is in a Special-Purpose Range

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Special-Purpose, Address Classification, Python, Networking, Security

Description: Build a comprehensive IPv6 address classifier that checks all IANA special-purpose ranges and returns the block name, properties, and appropriate usage guidance.

## Introduction

Applications that handle IPv6 addresses often need to classify them — is this address routable? Is it documentation-only? Is it loopback? A single, comprehensive classification function handles all these cases.

## Complete IPv6 Address Classifier

```python
import ipaddress
from dataclasses import dataclass
from typing import Optional

@dataclass
class IPv6AddressInfo:
    address: str
    block_name: str
    rfc: str
    source: bool
    destination: bool
    forwardable: bool
    globally_reachable: bool
    notes: str = ""

# Registry of special-purpose blocks
SPECIAL_PURPOSE_REGISTRY = [
    ("::1/128", "Loopback", "RFC 4291",
     True, True, False, False, "Host loopback"),
    ("::/128", "Unspecified", "RFC 4291",
     True, False, False, False, "Used before address assigned"),
    ("::ffff:0:0/96", "IPv4-Mapped", "RFC 4291",
     False, False, False, False, "Dual-stack socket API"),
    ("64:ff9b::/96", "NAT64 Well-Known", "RFC 6052",
     True, True, True, False, "NAT64 translation"),
    ("64:ff9b:1::/48", "NAT64 Local-Use", "RFC 8215",
     True, True, True, False, "Local NAT64 deployment"),
    ("100::/64", "Discard-Only", "RFC 6666",
     True, True, True, False, "Routing black holes"),
    ("2001::/32", "Teredo", "RFC 4380",
     True, True, True, False, "IPv4 NAT traversal (deprecated)"),
    ("2001:2::/48", "Benchmarking", "RFC 5180",
     True, True, True, False, "Network performance testing"),
    ("2001:20::/28", "ORCHIDv2", "RFC 7343",
     True, True, True, False, "Overlay cryptographic identifiers"),
    ("2001:db8::/32", "Documentation", "RFC 3849",
     False, False, False, False, "Examples and documentation ONLY"),
    ("2002::/16", "6to4", "RFC 3056",
     True, True, True, False, "IPv4-IPv6 tunnel (deprecated)"),
    ("3fff::/20", "Documentation (new)", "RFC 9637",
     False, False, False, False, "Additional documentation space"),
    ("5f00::/16", "SRv6 SIDs", "RFC 9602",
     True, True, True, True, "Segment Routing SIDs"),
    ("fc00::/7", "Unique-Local", "RFC 4193",
     True, True, True, False, "Private/organizational use"),
    ("fe80::/10", "Link-Local", "RFC 4291",
     True, True, False, False, "Single-link scope only"),
]

def classify_ipv6(addr_str: str) -> IPv6AddressInfo:
    """
    Classify an IPv6 address against all special-purpose ranges.
    Returns Global Unicast if no special range matches.
    """
    try:
        addr = ipaddress.IPv6Address(addr_str)
    except ValueError:
        raise ValueError(f"Invalid IPv6 address: {addr_str}")

    for prefix, name, rfc, src, dst, fwd, global_r, notes in SPECIAL_PURPOSE_REGISTRY:
        if addr in ipaddress.IPv6Network(prefix):
            return IPv6AddressInfo(
                address=str(addr),
                block_name=name,
                rfc=rfc,
                source=src,
                destination=dst,
                forwardable=fwd,
                globally_reachable=global_r,
                notes=notes
            )

    # Global unicast (not in any special range)
    return IPv6AddressInfo(
        address=str(addr),
        block_name="Global Unicast",
        rfc="RFC 4291",
        source=True, destination=True,
        forwardable=True, globally_reachable=True,
        notes="Standard globally routable address"
    )

# Test the classifier
test_addresses = [
    "::1", "::", "fe80::1", "2001:db8::1",
    "fc00::1", "64:ff9b::8.8.8.8", "5f00:1:1::1",
    "2001:4860:4860::8888"
]

for addr in test_addresses:
    info = classify_ipv6(addr)
    print(f"\n{addr}:")
    print(f"  Block: {info.block_name} ({info.rfc})")
    print(f"  Source:{info.source} Dst:{info.destination} "
          f"Fwd:{info.forwardable} Global:{info.globally_reachable}")
    print(f"  Notes: {info.notes}")
```

## Integration with Web Applications

```python
from flask import Flask, request, jsonify

app = Flask(__name__)

@app.route("/classify")
def classify():
    addr = request.args.get("addr", "")
    try:
        info = classify_ipv6(addr)
        return jsonify({
            "address": info.address,
            "block": info.block_name,
            "globally_reachable": info.globally_reachable,
            "safe_for_production": (
                info.globally_reachable and
                info.source and
                info.destination
            )
        })
    except ValueError as e:
        return jsonify({"error": str(e)}), 400
```

## Conclusion

A comprehensive IPv6 address classifier is essential for applications that handle user-supplied addresses. The Python implementation above covers all IANA special-purpose ranges and provides actionable metadata. Integrate this into your API input validation and use OneUptime to ensure your classification service remains healthy.
