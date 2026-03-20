# How to Use the IPv6 Flow Label Field

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Flow Label, QoS, Load Balancing, ECMP

Description: Understand the 20-bit IPv6 Flow Label field, how it enables per-flow QoS and stateless load balancing without deep packet inspection.

## Introduction

The IPv6 Flow Label is a 20-bit field unique to IPv6 with no IPv4 equivalent. Defined in RFC 6437, it allows a source to label packets belonging to the same flow, enabling routers and load balancers to provide special handling for that flow without inspecting upper-layer headers. While not yet universally used, the Flow Label is increasingly important for ECMP load balancing and QoS in high-speed networks.

## Flow Label Specification (RFC 6437)

```
Flow Label field: bits 12-31 of the first 32 bits of the IPv6 header
  Value 0x00000: not used / no specific flow request
  Non-zero:      identifies a specific flow (source-generated)

Requirements:
  - Source MUST use a consistent Flow Label for all packets in a flow
  - A "flow" = (Source Address, Destination Address, Flow Label)
  - Flow Label should be pseudo-random (not sequential)
  - Routers MUST NOT modify the Flow Label
  - Flow Label != 0 implies consistent 5-tuple (src, dst, protocol, src port, dst port)
```

## Generating Flow Labels

Sources should generate pseudo-random Flow Labels per flow:

```python
import hashlib
import struct
import socket

def generate_flow_label(src_addr: str, dst_addr: str,
                         src_port: int, dst_port: int,
                         protocol: int, secret: bytes = b"secret") -> int:
    """
    Generate a pseudo-random Flow Label for an IPv6 flow.
    Uses HMAC-based approach recommended by RFC 6437.

    Args:
        src_addr:  Source IPv6 address
        dst_addr:  Destination IPv6 address
        src_port:  Source port number
        dst_port:  Destination port number
        protocol:  IP protocol number (6=TCP, 17=UDP)
        secret:    Per-node secret key (rotated periodically)

    Returns:
        20-bit flow label (1 to 0xFFFFF)
    """
    # Build the 5-tuple
    src_bytes = socket.inet_pton(socket.AF_INET6, src_addr)
    dst_bytes = socket.inet_pton(socket.AF_INET6, dst_addr)
    ports = struct.pack("!HHB", src_port, dst_port, protocol)

    # Hash the 5-tuple with the secret
    data = src_bytes + dst_bytes + ports
    digest = hashlib.sha256(secret + data).digest()

    # Extract 20 bits from the hash
    flow_label = struct.unpack("!I", digest[:4])[0] & 0xFFFFF

    # RFC 6437: if result is 0, use 1 instead
    return flow_label if flow_label != 0 else 1

# Example
label = generate_flow_label(
    "2001:db8::1", "2001:db8::2",
    src_port=54321, dst_port=443, protocol=6
)
print(f"Flow Label: 0x{label:05X} ({label})")
```

## Setting Flow Label in Python Sockets

```python
import socket

# Linux supports setting the Flow Label via socket options
# Note: requires Linux kernel 3.6+

def create_socket_with_flow_label(flow_label: int) -> socket.socket:
    """Create an IPv6 socket with a specific Flow Label."""
    sock = socket.socket(socket.AF_INET6, socket.SOCK_STREAM)

    # IPV6_FLOWLABEL_MGR and IPV6_FLOWINFO_SEND
    # In Python, set via connect with explicit flowinfo parameter
    # The flow label is embedded in the flowinfo parameter of connect()
    # flowinfo = (flow_label & 0xFFFFF) | (traffic_class << 20)

    return sock, flow_label

# Connecting with a specific flow label
def connect_with_flow_label(dst_addr, dst_port, flow_label=0):
    sock = socket.socket(socket.AF_INET6, socket.SOCK_STREAM)
    # IPv6 connect() takes (host, port, flowinfo, scope_id)
    # flowinfo combines Traffic Class (bits 20-27) and Flow Label (bits 0-19)
    flowinfo = flow_label & 0xFFFFF
    sock.connect((dst_addr, dst_port, flowinfo, 0))
    return sock
```

## Flow Label for ECMP Load Balancing

The primary production use of Flow Labels is in ECMP (Equal-Cost Multi-Path) routing. Routers can hash on Flow Label instead of the full 5-tuple:

```bash
# Linux kernel: configure ECMP to include Flow Label in hash
# /etc/sysctl.conf
net.ipv6.flowlabel_state_ranges = 1

# Check current Flow Label settings
cat /proc/sys/net/ipv6/flowlabel_consistency
cat /proc/sys/net/ipv6/flowlabel_reflect
```

```
ECMP hash computation with Flow Label:

IPv4 typical ECMP hash: hash(src_ip, dst_ip, src_port, dst_port, protocol)
  → Requires parsing up to Layer 4

IPv6 with Flow Label: hash(src_ip, dst_ip, flow_label)
  → Only Layer 3 parsing needed
  → Works even with encrypted payloads (IPsec ESP)
  → Works with all protocols (not just TCP/UDP)
```

## Observing Flow Labels with tcpdump

```bash
# Display Flow Label for each captured IPv6 packet
sudo tcpdump -i eth0 -vv ip6 | grep "flow"

# Example output:
# 20:15:32 IP6 (hlim 64, next-header TCP (6), length 60)
#   2001:db8::1.54321 > 2001:db8::2.443: Flags [S]
#   (class 0x00, flowlabel 0x2a3b4)

# Filter packets with non-zero flow labels
sudo tcpdump -i eth0 "ip6 and (ip6[1:3] & 0x0fffff) != 0"
```

## Conclusion

The IPv6 Flow Label field enables routers and load balancers to identify and consistently handle flows without deep packet inspection. While its use is still evolving, its value in ECMP hashing (where it provides flow consistency even for encrypted traffic) and QoS (where it identifies flows that need special treatment) makes it an important tool for high-performance IPv6 network design. Sources should generate pseudo-random, per-flow labels using the 5-tuple hash approach recommended by RFC 6437.
