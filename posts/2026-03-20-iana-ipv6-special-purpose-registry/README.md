# How to Understand the IANA IPv6 Special-Purpose Address Registry

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, IANA, Special-Purpose Addresses, RFC 6890, Networking

Description: Understand the IANA IPv6 Special-Purpose Address Registry, which catalogues reserved address blocks with specific technical purposes distinct from globally routable unicast addresses.

## Introduction

The IANA Special-Purpose Address Registry (maintained at iana.org/assignments/iana-ipv6-special-registry) documents IPv6 address blocks that have specific, designated uses. Understanding this registry is essential for network operators, security teams, and application developers who need to correctly handle non-routable or special-function addresses.

## Registry Structure

Each entry in the registry has these properties:

| Property | Description |
|---|---|
| Address Block | The prefix |
| Name | Short name for the purpose |
| RFC | Defining RFC |
| Allocation Date | When allocated |
| Termination Date | If applicable |
| Source | Can be used as source? |
| Destination | Can be used as destination? |
| Forwardable | Should routers forward? |
| Globally Reachable | Reachable from public internet? |
| Reserved-by-Protocol | Reserved for protocol mechanism? |

## Major Special-Purpose Blocks

```python
# Special purpose blocks with key properties

SPECIAL_PURPOSE_BLOCKS = [
    {
        "prefix": "::1/128",
        "name": "IPv6 Loopback",
        "rfc": "RFC 4291",
        "source": True, "destination": True,
        "forwardable": False, "globally_reachable": False
    },
    {
        "prefix": "::/128",
        "name": "Unspecified",
        "rfc": "RFC 4291",
        "source": True, "destination": False,
        "forwardable": False, "globally_reachable": False
    },
    {
        "prefix": "::ffff:0:0/96",
        "name": "IPv4-Mapped",
        "rfc": "RFC 4291",
        "source": False, "destination": False,
        "forwardable": False, "globally_reachable": False
    },
    {
        "prefix": "64:ff9b::/96",
        "name": "NAT64 Well-Known",
        "rfc": "RFC 6052",
        "source": True, "destination": True,
        "forwardable": True, "globally_reachable": False
    },
    {
        "prefix": "100::/64",
        "name": "Discard-Only",
        "rfc": "RFC 6666",
        "source": True, "destination": True,
        "forwardable": True, "globally_reachable": False
    },
    {
        "prefix": "2001::/32",
        "name": "Teredo",
        "rfc": "RFC 4380",
        "source": True, "destination": True,
        "forwardable": True, "globally_reachable": False
    },
    {
        "prefix": "2001:2::/48",
        "name": "Benchmarking",
        "rfc": "RFC 5180",
        "source": True, "destination": True,
        "forwardable": True, "globally_reachable": False
    },
    {
        "prefix": "2001:db8::/32",
        "name": "Documentation",
        "rfc": "RFC 3849",
        "source": False, "destination": False,
        "forwardable": False, "globally_reachable": False
    },
    {
        "prefix": "2002::/16",
        "name": "6to4",
        "rfc": "RFC 3056",
        "source": True, "destination": True,
        "forwardable": True, "globally_reachable": False
    },
    {
        "prefix": "5f00::/16",
        "name": "SRv6 SIDs",
        "rfc": "RFC 9602",
        "source": True, "destination": True,
        "forwardable": True, "globally_reachable": True
    },
    {
        "prefix": "fc00::/7",
        "name": "Unique-Local",
        "rfc": "RFC 4193",
        "source": True, "destination": True,
        "forwardable": True, "globally_reachable": False
    },
    {
        "prefix": "fe80::/10",
        "name": "Link-Local",
        "rfc": "RFC 4291",
        "source": True, "destination": True,
        "forwardable": False, "globally_reachable": False
    },
]
```

## Checking Addresses Against the Registry

```python
import ipaddress

def classify_ipv6_address(addr_str: str) -> str:
    """Check if an IPv6 address falls in a special-purpose range."""
    addr = ipaddress.IPv6Address(addr_str)

    checks = [
        (ipaddress.IPv6Network("::1/128"), "Loopback"),
        (ipaddress.IPv6Network("::/128"), "Unspecified"),
        (ipaddress.IPv6Network("::ffff:0:0/96"), "IPv4-Mapped"),
        (ipaddress.IPv6Network("64:ff9b::/96"), "NAT64 (Well-Known)"),
        (ipaddress.IPv6Network("64:ff9b:1::/48"), "NAT64 (Local-Use)"),
        (ipaddress.IPv6Network("100::/64"), "Discard-Only"),
        (ipaddress.IPv6Network("2001::/32"), "Teredo"),
        (ipaddress.IPv6Network("2001:2::/48"), "Benchmarking"),
        (ipaddress.IPv6Network("2001:db8::/32"), "Documentation"),
        (ipaddress.IPv6Network("2002::/16"), "6to4"),
        (ipaddress.IPv6Network("5f00::/16"), "SRv6 SIDs"),
        (ipaddress.IPv6Network("fc00::/7"), "Unique-Local"),
        (ipaddress.IPv6Network("fe80::/10"), "Link-Local"),
    ]

    for network, name in checks:
        if addr in network:
            return name

    return "Global Unicast"

# Test
for test_addr in ["::1", "fe80::1", "2001:db8::1", "2001:4860:4860::8888"]:
    print(f"{test_addr}: {classify_ipv6_address(test_addr)}")
```

## Why This Registry Matters

1. **Security filtering**: Firewalls and routers should reject certain special-purpose addresses at network boundaries
2. **Application validation**: Web apps should not store or route to non-globally-reachable addresses
3. **Documentation**: Use `2001:db8::/32` in all examples, never real addresses
4. **Monitoring**: Don't alert on traffic to/from special-purpose ranges that are expected

## Conclusion

The IANA IPv6 Special-Purpose Address Registry is the authoritative reference for IPv6 address semantics. Applications, firewalls, and monitoring systems should consult it when classifying addresses. The Python function above provides a programmatic classification suitable for use in network tooling and security systems monitored by OneUptime.
