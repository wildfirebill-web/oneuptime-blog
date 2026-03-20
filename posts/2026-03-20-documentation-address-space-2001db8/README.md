# How to Use the Documentation Address Space (2001:db8::/32 and 3fff::/20)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Documentation, 2001:db8, 3fff, RFC 3849, Networking

Description: Understand the IPv6 documentation address spaces 2001:db8::/32 and 3fff::/20, their proper usage in examples and documentation, and how to avoid them in production.

## Introduction

RFC 3849 reserves `2001:db8::/32` for use in documentation, books, and examples - similar to IPv4's `192.0.2.0/24`. RFC 9637 allocates the additional `3fff::/20` block for documentation purposes. These addresses must never appear in production configurations.

## Usage in Examples

```bash
# CORRECT: Use 2001:db8:: in all documentation/examples

ping6 2001:db8::1
ssh 2001:db8::server
curl http://[2001:db8::1]/

# WRONG: Use real addresses in documentation
# This exposes operational details and may confuse readers
ping6 2a00:1450:4007:818::200e  # Don't use Google's real address
```

```python
# Code examples should use 2001:db8::/32 or 3fff::/20
EXAMPLE_SERVER_IPv6 = "2001:db8::1"      # Server address
EXAMPLE_CLIENT_IPv6 = "2001:db8::100"    # Client address
EXAMPLE_SUBNET = "2001:db8:1::/64"       # Subnet

# IPv6 subnets for documentation
EXAMPLE_PREFIX_1 = "2001:db8:1::/64"
EXAMPLE_PREFIX_2 = "2001:db8:2::/64"
EXAMPLE_BGP_PREFIX = "2001:db8::/32"
```

## Detecting Documentation Addresses in Production

```python
import ipaddress

def uses_documentation_address(addr: str) -> bool:
    """Detect if an address is from the documentation space."""
    doc_prefixes = [
        "2001:db8::/32",
        "3fff::/20",
    ]
    try:
        a = ipaddress.IPv6Address(addr)
        for prefix in doc_prefixes:
            if a in ipaddress.IPv6Network(prefix):
                return True
    except ValueError:
        pass
    return False

# Use this in config validators
def validate_production_config(config: dict) -> list:
    """Return warnings for any documentation addresses in config."""
    warnings = []
    for key, value in config.items():
        if isinstance(value, str) and ':' in value:
            if uses_documentation_address(value):
                warnings.append(
                    f"WARNING: {key}={value} uses documentation address"
                )
    return warnings
```

## Filtering Documentation Addresses

```bash
# Documentation addresses should never appear in production traffic
ip6tables -A INPUT -s 2001:db8::/32 -j DROP
ip6tables -A INPUT -s 3fff::/20 -j DROP
ip6tables -A FORWARD -s 2001:db8::/32 -j DROP
ip6tables -A FORWARD -s 3fff::/20 -j DROP
```

## The New 3fff::/20 Block

RFC 9637 (2024) allocates `3fff::/20` as an additional documentation space because many tools hard-coded `2001:db8::/32` checks and the larger block allows more diverse example addressing.

```python
DOCUMENTATION_PREFIXES = [
    "2001:db8::/32",  # RFC 3849 (original)
    "3fff::/20",      # RFC 9637 (additional, 2024)
]
```

## Conclusion

Always use `2001:db8::/32` or `3fff::/20` in documentation, blog posts, and code examples. Filter these addresses from production traffic. Use a config validator that detects documentation addresses before applying changes to production infrastructure monitored by OneUptime.
