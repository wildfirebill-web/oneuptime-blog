# How to Identify Reserved IPv4 Addresses

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv4, Networking, Reserved Addresses, IP Addressing, RFC, Network Design

Description: Several IPv4 address blocks are reserved by IANA for special purposes including loopback, link-local, documentation, multicast, and future use, and these must never be assigned to regular hosts or...

## IANA Special-Purpose Address Registry

| Block | Purpose | Reference |
|-------|---------|-----------|
| 0.0.0.0/8 | "This" network (source only) | RFC 1122 |
| 10.0.0.0/8 | Private use | RFC 1918 |
| 100.64.0.0/10 | Shared address space (CGNAT) | RFC 6598 |
| 127.0.0.0/8 | Loopback | RFC 5735 |
| 169.254.0.0/16 | Link-local (APIPA) | RFC 3927 |
| 172.16.0.0/12 | Private use | RFC 1918 |
| 192.0.0.0/24 | IETF Protocol Assignments | RFC 5736 |
| 192.0.2.0/24 | TEST-NET-1 (documentation) | RFC 5737 |
| 192.168.0.0/16 | Private use | RFC 1918 |
| 198.18.0.0/15 | Benchmarking | RFC 2544 |
| 198.51.100.0/24 | TEST-NET-2 (documentation) | RFC 5737 |
| 203.0.113.0/24 | TEST-NET-3 (documentation) | RFC 5737 |
| 224.0.0.0/4 | Multicast | RFC 5771 |
| 240.0.0.0/4 | Reserved (Experimental) | RFC 1112 |
| 255.255.255.255/32 | Limited broadcast | RFC 919 |

## Python: Checking for Reserved Addresses

```python
import ipaddress

def describe_address(ip: str) -> str:
    """Return a description of the address type."""
    addr = ipaddress.IPv4Address(ip)
    if addr.is_loopback:
        return "Loopback"
    if addr.is_link_local:
        return "Link-local (APIPA)"
    if addr.is_private:
        return "Private (RFC 1918)"
    if addr.is_multicast:
        return "Multicast"
    if addr.is_reserved:
        return "Reserved"
    if addr.is_unspecified:
        return "Unspecified (0.0.0.0)"
    if addr.is_global:
        return "Global (public)"
    return "Unknown"

# Test a variety of addresses

for ip in ["0.0.0.0", "10.1.2.3", "100.64.0.1", "127.0.0.1",
           "169.254.1.1", "192.0.2.1", "224.0.0.1", "240.0.0.1",
           "255.255.255.255", "8.8.8.8"]:
    print(f"{ip:18s} -> {describe_address(ip)}")
```

## Documentation Addresses (192.0.2.0/24, 198.51.100.0/24, 203.0.113.0/24)

These three /24 blocks (TEST-NET-1, TEST-NET-2, TEST-NET-3) are specifically reserved for use in examples, documentation, books, and tutorials. They must never be assigned to real hosts:

```python
# Always use TEST-NET addresses in documentation, not real ones
EXAMPLE_HOSTS = [
    "192.0.2.1",   # TEST-NET-1 host
    "198.51.100.50", # TEST-NET-2 host
    "203.0.113.10",  # TEST-NET-3 host
]
```

## Benchmarking Addresses (198.18.0.0/15)

The 198.18.0.0/15 block is reserved for network device benchmarking (RFC 2544). These addresses should only appear in controlled benchmark environments.

## Key Takeaways

- Never assign documentation addresses (192.0.2.0/24, 198.51.100.0/24, 203.0.113.0/24) to real hosts.
- Filter reserved source addresses at network borders to prevent spoofing.
- Use Python's `ipaddress` module flags (`is_private`, `is_loopback`, etc.) to classify addresses programmatically.
- The 240.0.0.0/4 block (Class E) remains reserved and is not routed on the internet.
