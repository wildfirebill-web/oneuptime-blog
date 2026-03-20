# How to Check If an IPv6 Address Is in a Special-Purpose Range

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Special-Purpose, Classification, Python, Validation, IANA

Description: Build a comprehensive IPv6 address classifier that checks against all IANA special-purpose ranges and returns the address type and its properties.

## Introduction

Checking whether an IPv6 address falls in a special-purpose range is a common requirement for security filters, logging systems, and network tools. The IANA Special-Purpose Address Registry documents all reserved prefixes. This post provides a complete Python implementation for classifying any IPv6 address.

## Complete IPv6 Address Classifier

```python
import ipaddress
from dataclasses import dataclass
from typing import Optional

@dataclass
class IPv6AddressInfo:
    address: str
    category: str
    rfc: str
    source: bool
    destination: bool
    forwardable: bool
    globally_reachable: bool
    description: str

SPECIAL_PURPOSE_REGISTRY = [
    # (prefix, category, rfc, source, dest, forwardable, globally_reachable, description)
    ("::1/128",          "Loopback",        "RFC 4291", True,  True,  False, False, "IPv6 loopback address"),
    ("::/128",           "Unspecified",     "RFC 4291", True,  False, False, False, "Unspecified address"),
    ("::ffff:0:0/96",    "IPv4-Mapped",     "RFC 4291", False, False, False, False, "IPv4-mapped IPv6 addresses"),
    ("::ffff:0:0:0/96",  "IPv4-Translated", "RFC 2765", False, False, False, False, "IPv4-translated addresses"),
    ("64:ff9b::/96",     "NAT64-WKP",       "RFC 6052", True,  True,  True,  False, "NAT64 Well-Known Prefix"),
    ("64:ff9b:1::/48",   "NAT64-Local",     "RFC 8215", True,  True,  True,  False, "NAT64 Local-Use Prefix"),
    ("100::/64",         "Discard-Only",    "RFC 6666", True,  True,  True,  False, "Discard-only address block"),
    ("2001::/32",        "Teredo",          "RFC 4380", True,  True,  True,  False, "Teredo tunneling"),
    ("2001:2::/48",      "Benchmarking",    "RFC 5180", True,  True,  True,  False, "Benchmarking address space"),
    ("2001:3::/32",      "AMT",             "RFC 7450", True,  True,  True,  True,  "Automatic Multicast Tunneling"),
    ("2001:db8::/32",    "Documentation",   "RFC 3849", False, False, False, False, "Documentation prefix"),
    ("2001:20::/28",     "ORCHIDv2",        "RFC 7343", False, False, False, False, "Overlay Routable Cryptographic Hash IDs"),
    ("2002::/16",        "6to4",            "RFC 3056", True,  True,  True,  False, "6to4 addressing"),
    ("3fff::/20",        "Documentation",   "RFC 9637", False, False, False, False, "Documentation prefix (new)"),
    ("5f00::/16",        "SRv6-SID",        "RFC 9602", True,  True,  True,  True,  "SRv6 Segment Identifiers"),
    ("fc00::/7",         "Unique-Local",    "RFC 4193", True,  True,  True,  False, "Unique Local Addresses"),
    ("fe80::/10",        "Link-Local",      "RFC 4291", True,  True,  False, False, "Link-local addresses"),
    ("ff00::/8",         "Multicast",       "RFC 4291", False, True,  True,  False, "Multicast addresses"),
]

def classify_ipv6(addr_str: str) -> IPv6AddressInfo:
    """Classify an IPv6 address against the IANA Special-Purpose Registry."""
    try:
        addr = ipaddress.IPv6Address(addr_str)
    except ValueError as e:
        raise ValueError(f"Invalid IPv6 address: {addr_str}") from e

    for prefix, category, rfc, src, dst, fwd, global_r, desc in SPECIAL_PURPOSE_REGISTRY:
        network = ipaddress.IPv6Network(prefix)
        if addr in network:
            return IPv6AddressInfo(
                address=str(addr),
                category=category,
                rfc=rfc,
                source=src,
                destination=dst,
                forwardable=fwd,
                globally_reachable=global_r,
                description=desc,
            )

    # Not in any special-purpose range
    return IPv6AddressInfo(
        address=str(addr),
        category="Global Unicast",
        rfc="RFC 4291",
        source=True,
        destination=True,
        forwardable=True,
        globally_reachable=True,
        description="Global unicast address",
    )

# Test the classifier
test_addresses = [
    "::1",
    "::",
    "64:ff9b::808:808",
    "100::1",
    "2001::1",
    "2001:db8::1",
    "3fff::1",
    "5f00:1:0:e001::",
    "fc00::1",
    "fd00:1:2:3::4",
    "fe80::1",
    "ff02::1",
    "2001:4860:4860::8888",
]

for addr in test_addresses:
    info = classify_ipv6(addr)
    print(f"{addr:40s} → {info.category} ({info.rfc})")
```

## Batch Classification

```python
def classify_from_log(log_file: str) -> dict:
    """
    Classify all IPv6 addresses found in a log file.
    Returns a summary count by category.
    """
    import re
    from collections import Counter

    ipv6_pattern = re.compile(
        r'(?:[0-9a-fA-F]{0,4}:){2,7}[0-9a-fA-F]{0,4}'
    )
    category_counts = Counter()

    with open(log_file) as f:
        for line in f:
            for match in ipv6_pattern.finditer(line):
                try:
                    info = classify_ipv6(match.group())
                    category_counts[info.category] += 1
                except ValueError:
                    pass

    return dict(category_counts.most_common())
```

## Conclusion

A complete IANA registry-based classifier lets you quickly determine what any IPv6 address is and whether it should appear in logs, configurations, or traffic. Integrate this classifier into your log analysis pipelines, security tooling, and network automation to detect misconfigured or unexpected addresses. Use OneUptime to alert when special-purpose addresses appear in unexpected contexts.
