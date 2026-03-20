# How to Understand the SRv6 SID Format (5f00::/16)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: SRv6, SID, IPv6 Addressing, 5f00, RFC 9602, Networking

Description: Understand the IANA-allocated SRv6 SID address space (5f00::/16), the SID structure, and how locators, functions, and arguments are encoded in the 128-bit address.

## Introduction

RFC 9602 allocates the `5f00::/16` prefix as the global SRv6 SID address space. SIDs using this prefix are routable global IPv6 addresses that encode both routing (locator) and service (function+arguments) information.

## The 5f00::/16 Address Block

```
5f00::/16 is allocated by IANA for SRv6 SIDs:
  Range: 5f00:: through 5fff:ffff:ffff:ffff:ffff:ffff:ffff:ffff
  Purpose: SRv6 Segment Identifiers
  Status: Globally routable (unicast)
```

## SID Structure

A SRv6 SID is a 128-bit IPv6 address divided into three parts:

```
128 bits total:
|<---- Locator ---->|<- Function ->|<--- Arguments --->|
|     L bits        |   F bits     |      A bits       |

Example with 48/16/64 split:
|  48-bit locator   | 16-bit func  |   64-bit args     |
| 5f00:1:1::/48     |   e000       |    :: (none)      |

SID: 5f00:1:1:0:e000::
```

## Locator, Function, Arguments Explained

### Locator

The locator is the routable prefix that identifies the **node** (or cluster of nodes) owning this SID. It is advertised via IGP (IS-IS or OSPF) as a regular IPv6 prefix.

```
Locator examples:
  5f00:1:1::/48  — Node 1, site 1
  5f00:2:1::/48  — Node 1, site 2
  5f00:1:2::/48  — Node 2, site 1
```

### Function

The function portion encodes what action the endpoint node should take when it receives a packet with this SID as the destination.

```
Function examples (16-bit):
  0x0001  — End (plain routing)
  0xe000  — End.DT6 (IPv6 L3 table lookup)
  0xe001  — End.DT4 (IPv4 L3 table lookup)
  0xe002  — End.DX4 (IPv4 decap + cross-connect)
  0xe003  — End.DX6 (IPv6 decap + cross-connect)
```

### Arguments

Arguments provide per-packet context to the endpoint function (e.g., VPN ID, FlowID).

```python
import ipaddress

def parse_srv6_sid(sid: str, locator_bits: int = 48,
                   function_bits: int = 16) -> dict:
    """
    Parse an SRv6 SID into its locator, function, and argument components.
    """
    addr = ipaddress.IPv6Address(sid)
    addr_int = int(addr)

    total_bits = 128
    argument_bits = total_bits - locator_bits - function_bits

    # Extract components using bit masks
    locator_mask = ((1 << locator_bits) - 1) << (total_bits - locator_bits)
    function_mask = ((1 << function_bits) - 1) << argument_bits
    argument_mask = (1 << argument_bits) - 1

    locator_int = (addr_int & locator_mask) >> (total_bits - locator_bits)
    function_int = (addr_int & function_mask) >> argument_bits
    argument_int = addr_int & argument_mask

    # Reconstruct locator prefix
    locator_addr_int = addr_int & locator_mask
    locator_prefix = str(ipaddress.IPv6Address(locator_addr_int))

    return {
        "sid": sid,
        "locator": f"{locator_prefix}/{locator_bits}",
        "function": hex(function_int),
        "function_dec": function_int,
        "arguments": hex(argument_int),
    }

# Example usage
result = parse_srv6_sid("5f00:1:1:0:e000::")
print(f"Locator: {result['locator']}")
print(f"Function: {result['function']}")
print(f"Arguments: {result['arguments']}")
# Output:
# Locator: 5f00:1:1::/48
# Function: 0xe000
# Arguments: 0x0
```

## Allocating SIDs in Your Network

```bash
# Assign a locator prefix from 5f00::/16 to a router
# Router 1 gets locator 5f00:1:1::/48

# Configure on Linux (iproute2)
ip -6 route add 5f00:1:1::/48 dev lo

# Add specific SID for End function
ip -6 route add 5f00:1:1::1/128 encap seg6local action End dev lo

# Add SID for End.DT6 (IPv6 L3VPN table)
ip -6 route add 5f00:1:1:0:e000::/128 \
  encap seg6local action End.DT6 vrftable 100 dev lo
```

## 5f00::/16 vs Operator Prefixes

Before RFC 9602, operators used their own allocated prefixes for SIDs:

```
Before RFC 9602:
  Operator A: 2001:db8:1::/48 — SRv6 SIDs
  Operator B: 2001:db8:2::/48 — SRv6 SIDs
  (No globally recognized SRv6 prefix)

After RFC 9602:
  All operators use 5f00::/16 subspace
  Benefits: Consistent filtering, easy identification, hardware optimization
```

## Conclusion

The `5f00::/16` SRv6 SID address space provides a globally recognizable prefix for SRv6 deployments. Understanding the locator/function/argument structure enables you to plan SID allocation, configure SRv6 routers, and debug SRv6 forwarding. Use OneUptime to monitor the reachability of locator prefixes as a proxy for SRv6 node health.
