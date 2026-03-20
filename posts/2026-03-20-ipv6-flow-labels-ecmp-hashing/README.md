# How to Use Flow Labels for ECMP Hashing

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, ECMP, Flow Label, BGP, Routing

Description: Use IPv6 Flow Labels in Equal-Cost Multi-Path routing to ensure consistent per-flow path selection without deep packet inspection of encrypted traffic.

## Introduction

ECMP (Equal-Cost Multi-Path) routing allows traffic to be distributed across multiple equal-cost paths. The challenge is ensuring that packets belonging to the same TCP flow always take the same path — otherwise, out-of-order delivery causes performance problems. IPv6 Flow Labels solve this elegantly: by hashing on the Flow Label (which is constant for a given flow), ECMP routers can make consistent path decisions without inspecting TCP ports or payloads.

## The ECMP Problem

```
Without consistent hashing:
Flow A: Packet 1 → Path 1, Packet 2 → Path 2 (different latencies)
Result: TCP out-of-order → retransmissions → degraded throughput

With Flow Label ECMP hashing:
Flow A: Flow Label = 0x2A3B4
→ hash(src, dst, 0x2A3B4) → always maps to Path 1
Result: All packets take same path → in-order delivery
```

## Linux Kernel ECMP Configuration

```bash
# Linux ECMP uses 5-tuple or flow label for hashing
# Check current ECMP hash policy
cat /proc/sys/net/ipv6/flowlabel_state_ranges

# Enable flow-label-aware ECMP
sudo sysctl -w net.ipv6.flowlabel_state_ranges=1

# Set up ECMP routes
ip -6 route add 2001:db8:server::/48 \
    nexthop via 2001:db8:transit1::1 dev eth0 weight 1 \
    nexthop via 2001:db8:transit2::1 dev eth1 weight 1 \
    nexthop via 2001:db8:transit3::1 dev eth2 weight 1

# Verify ECMP is active
ip -6 route show 2001:db8:server::/48
# Should show 3 nexthops
```

## ECMP Hash Algorithms

```python
import hashlib
import socket
import struct

def ecmp_hash(src_addr: str, dst_addr: str, flow_label: int,
              num_paths: int) -> int:
    """
    Compute the ECMP path selection for an IPv6 flow.

    Uses (src, dst, flow_label) for consistent per-flow hashing.
    Falls back to 5-tuple when flow_label=0.

    Returns: path index (0 to num_paths-1)
    """
    src = socket.inet_pton(socket.AF_INET6, src_addr)
    dst = socket.inet_pton(socket.AF_INET6, dst_addr)

    if flow_label != 0:
        # Use 3-tuple hash when flow label is set
        hash_input = src + dst + struct.pack("!I", flow_label)
        method = "3-tuple (flow label)"
    else:
        # Fall back to address-only hash (no port info for simple example)
        hash_input = src + dst
        method = "2-tuple (no flow label)"

    hash_val = int(hashlib.sha256(hash_input).hexdigest(), 16)
    path = hash_val % num_paths

    print(f"  Hash method: {method}")
    print(f"  Path selected: {path} of {num_paths}")
    return path

# Simulate ECMP path selection for multiple flows
print("ECMP Path Selection Simulation")
print("="*50)

flows = [
    ("2001:db8:client::1", "2001:db8:server::1", 0x12345),
    ("2001:db8:client::1", "2001:db8:server::1", 0x67890),  # Same src/dst, diff flow
    ("2001:db8:client::2", "2001:db8:server::1", 0x12345),  # Diff src, same flow label
    ("2001:db8:client::1", "2001:db8:server::1", 0),         # No flow label
]

for src, dst, fl in flows:
    print(f"\nFlow: {src} → {dst} (FL: 0x{fl:05X})")
    path = ecmp_hash(src, dst, fl, num_paths=3)
```

## Cisco IOS-XR ECMP with Flow Label

```
# Cisco IOS-XR: configure ECMP hash to include flow label
router cef
  load-balance algorithm tunnel src-dst-ip-gre-key-port-proto-tunnel-id

# For IPv6 specifically, the flow label is included in hash
# when 'ipv6 cef load-sharing algorithm include-ports source destination'
# is configured:
ipv6 cef load-sharing algorithm include-ports source destination
```

## Juniper Junos ECMP Configuration

```
# Juniper: include Flow Label in ECMP hash
set forwarding-options hash-key family inet6 layer-3 source-address
set forwarding-options hash-key family inet6 layer-3 destination-address
set forwarding-options hash-key family inet6 flow-label

# This ensures (src, dst, flow_label) are all included in the ECMP hash
```

## Generating Good Flow Labels for ECMP

Sources should generate diverse, pseudo-random Flow Labels to ensure good load distribution:

```python
import os
import hashlib
import struct
import socket

# RFC 6437 recommended approach: hash 5-tuple with rotating secret
class FlowLabelGenerator:
    def __init__(self):
        # Secret rotated periodically to prevent tracking
        self._secret = os.urandom(16)

    def generate(self, src: str, dst: str, src_port: int,
                 dst_port: int, protocol: int) -> int:
        """Generate a flow label for a given 5-tuple."""
        src_bytes = socket.inet_pton(socket.AF_INET6, src)
        dst_bytes = socket.inet_pton(socket.AF_INET6, dst)
        ports = struct.pack("!HHB", src_port, dst_port, protocol)

        digest = hashlib.sha256(self._secret + src_bytes + dst_bytes + ports).digest()
        flow_label = struct.unpack("!I", digest[:4])[0] & 0xFFFFF
        return flow_label if flow_label != 0 else 1

gen = FlowLabelGenerator()

# Show distribution across 3 ECMP paths
from collections import Counter
paths = []
for i in range(10000):
    fl = gen.generate("2001:db8::1", "2001:db8::2", i, 443, 6)
    path = fl % 3
    paths.append(path)

distribution = Counter(paths)
for path, count in sorted(distribution.items()):
    print(f"Path {path}: {count} flows ({count/100:.1f}%)")
```

## Conclusion

IPv6 Flow Labels improve ECMP implementations by providing a stable, per-flow identifier at Layer 3. Network devices can hash on (src, dst, flow_label) without inspecting TCP ports or payloads, enabling efficient ECMP even for IPsec-encrypted traffic. Sources should generate pseudo-random Flow Labels using the RFC 6437 algorithm to ensure even load distribution across ECMP paths.
