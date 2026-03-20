# How to Calculate Wildcard Masks from Subnet Masks

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv4, Wildcard Mask, Subnetting, ACL, OSPF, Networking

Description: A wildcard mask is the bitwise inverse of a subnet mask, with 0 bits indicating matching positions and 1 bits indicating 'don't care' positions, used in ACLs and routing protocol configurations.

## What Is a Wildcard Mask?

While subnet masks use 1s for network bits and 0s for host bits, wildcard masks invert this:
- **0** = must match
- **1** = don't care (any value)

To compute a wildcard mask: subtract each octet of the subnet mask from 255.

```text
Subnet mask: 255.255.255.0
Wildcard:    255-255 . 255-255 . 255-255 . 255-0 = 0.0.0.255
```

## Common Conversions

| Subnet Mask | CIDR | Wildcard Mask |
|-------------|------|--------------|
| 255.0.0.0 | /8 | 0.255.255.255 |
| 255.255.0.0 | /16 | 0.0.255.255 |
| 255.255.255.0 | /24 | 0.0.0.255 |
| 255.255.255.128 | /25 | 0.0.0.127 |
| 255.255.255.192 | /26 | 0.0.0.63 |
| 255.255.255.240 | /28 | 0.0.0.15 |
| 255.255.255.252 | /30 | 0.0.0.3 |

## Python Calculation

```python
import socket, struct

def subnet_to_wildcard(mask: str) -> str:
    """Convert a dotted-decimal subnet mask to a wildcard mask."""
    mask_int = struct.unpack("!I", socket.inet_aton(mask))[0]
    wildcard_int = ~mask_int & 0xFFFFFFFF
    return socket.inet_ntoa(struct.pack("!I", wildcard_int))

def cidr_to_wildcard(prefix: int) -> str:
    """Convert CIDR prefix length directly to wildcard mask."""
    # Wildcard = NOT subnet_mask
    wildcard_int = (1 << (32 - prefix)) - 1
    return socket.inet_ntoa(struct.pack("!I", wildcard_int))

# Examples

for mask in ["255.255.255.0", "255.255.240.0", "255.255.255.252"]:
    print(f"Subnet {mask:16s} -> Wildcard {subnet_to_wildcard(mask)}")

for prefix in [8, 16, 24, 26, 28, 30]:
    print(f"/{prefix:2d} -> Wildcard {cidr_to_wildcard(prefix)}")
```

## Using ipaddress Module

```python
import ipaddress

for cidr in ["10.0.0.0/8", "192.168.1.0/24", "172.16.0.0/20"]:
    net = ipaddress.IPv4Network(cidr)
    print(f"{cidr:20s}  mask={net.netmask}  wildcard={net.hostmask}")
```

## Key Takeaways

- Wildcard mask = 255.255.255.255 − subnet mask (bitwise NOT).
- 0 bits in wildcard = must match; 1 bits = don't care.
- Use `ipaddress.IPv4Network(cidr).hostmask` for quick lookups.
- Wildcard masks appear in Cisco ACLs, OSPF network statements, and NAT configurations.
