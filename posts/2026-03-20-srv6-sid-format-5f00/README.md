# How to Understand the SRv6 SID Format (5f00::/16)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: SRv6, SID, 5f00, RFC 9602, IPv6, Segment Routing

Description: Understand the SRv6 SID address format using the 5f00::/16 prefix, locator structure, function encoding, and how SIDs are constructed and allocated.

## Introduction

RFC 9602 allocates `5f00::/16` as the globally routable address space for SRv6 Segment Identifiers (SIDs). A SID is a 128-bit IPv6 address that encodes both a locator (identifying the node) and a function (the behavior to invoke at that node). Understanding the SID format is essential for planning, configuring, and troubleshooting SRv6 deployments.

## SID Structure

```
 | <--- Locator (N bits) ---> | <-- Function (F bits) --> | <- Args (A bits) -> |
 |                            |                           |                     |
 |  Block  |   Node ID        |  Function Code            |    Arguments        |
 | 16 bits |  variable        |  variable                 |    variable         |

 Total = 128 bits = Locator + Function + Arguments

 Typical allocation:
   Block:    16 bits  (5f00::/16 from RFC 9602)
   Node ID:  32 bits  (identifies specific router)
   Function: 16 bits  (behavior at that node)
   Args:     64 bits  (optional, for stateless parameters)

 Example:
   5f00:0001:0000:e001:0000:0000:0000:0000/128
   Block=5f00, Node=0001:0000, Function=e001, Args=0000:...
   Written compressed: 5f00:1:0:e001::
```

## Locator Structure and Allocation

```bash
# Example locator plan for a 3-node network
# Each node gets a /48 locator within 5f00::/16

NODE_R1_LOCATOR="5f00:1::/48"
NODE_R2_LOCATOR="5f00:2::/48"
NODE_R3_LOCATOR="5f00:3::/48"

# Assign locator to loopback
ip -6 addr add 5f00:1::/128 dev lo      # Loopback SID (End function)
ip -6 addr add 5f00:1:0:e001::/128 dev lo  # End.X SID (function e001)
ip -6 addr add 5f00:1:0:e002::/128 dev lo  # End.X SID to different next-hop

# Enable SRv6 on interface
sysctl -w net.ipv6.conf.eth0.seg6_enabled=1
sysctl -w net.ipv6.conf.all.seg6_enabled=1
```

## Well-Known Function Codes

```
Function 0x0001 = End
Function 0x0002 = End.X
Function 0x0003 = End.T
Function 0x0004 = End.DX2
Function 0x0005 = End.DX2V
Function 0x0006 = End.DT2U
Function 0x0007 = End.DT2M
Function 0x000B = End.DX6
Function 0x000C = End.DX4
Function 0x000D = End.DT6
Function 0x000E = End.DT4
Function 0x000F = End.DT46
Function 0x0010 = End.B6.Encaps
Function 0x001C = End.BM

Note: Well-known functions are below 0x8000.
      Locally significant functions are 0x8000-0xFFFF.
```

## SID Construction in Python

```python
import ipaddress

def build_srv6_sid(block: str, node_id: int, function: int, args: int = 0) -> str:
    """
    Build an SRv6 SID from components.
    block: e.g. "5f00"
    node_id: 32-bit node identifier
    function: 16-bit function code
    args: 64-bit arguments (default 0)
    """
    # Pack into 128-bit integer
    # Layout: block(16) | node(32) | function(16) | args(64)
    block_int = int(block, 16)
    sid_int = (
        (block_int << 112) |
        (node_id << 80) |
        (function << 64) |
        args
    )
    return str(ipaddress.IPv6Address(sid_int))

# Examples
print(build_srv6_sid("5f00", 1, 0xe001))   # 5f00:1:0:e001::
print(build_srv6_sid("5f00", 2, 0xe000))   # 5f00:2:0:e000::
print(build_srv6_sid("5f00", 3, 0x0001))   # 5f00:3:0:1::  (End)

def parse_srv6_sid(sid: str) -> dict:
    """Parse an SRv6 SID into components."""
    addr_int = int(ipaddress.IPv6Address(sid))
    return {
        "block":    hex((addr_int >> 112) & 0xFFFF),
        "node_id":  hex((addr_int >> 80) & 0xFFFFFFFF),
        "function": hex((addr_int >> 64) & 0xFFFF),
        "args":     hex(addr_int & 0xFFFFFFFFFFFFFFFF),
    }

print(parse_srv6_sid("5f00:1:0:e001::"))
# {'block': '0x5f00', 'node_id': '0x10000', 'function': '0xe001', 'args': '0x0'}
```

## Advertising SIDs via IS-IS or BGP

```bash
# FRR IS-IS: advertise SRv6 locator
# /etc/frr/frr.conf
# router isis CORE
#   segment-routing srv6
#    locator MAIN
#     prefix 5f00:1::/48
```

## Conclusion

The `5f00::/16` address space provides globally routable SRv6 SIDs. Each SID encodes a locator (node identity) and function (behavior). Plan your SID allocation with a clear locator hierarchy. Use the Python parsing functions above to validate SIDs in automation scripts. Monitor SID reachability with OneUptime to ensure the control plane is advertising all required segments.
