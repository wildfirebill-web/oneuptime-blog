# How to Use Flow Labels for Stateless Load Balancing

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Flow Label, Load Balancing, ECMP, Networking

Description: Use IPv6 Flow Labels to implement stateless per-flow load balancing across multiple uplinks or servers without maintaining per-connection state tables.

## Introduction

IPv6 Flow Labels enable stateless load balancing by providing a per-flow identifier at Layer 3 without requiring the load balancer to inspect TCP/UDP port numbers. This is particularly valuable for encrypted traffic (IPsec, QUIC, TLS) where port numbers may be obscured or unavailable. A load balancer can hash on (Source IP, Destination IP, Flow Label) and always send the same flow to the same backend.

## Why Flow Labels Help Load Balancing

```
Traditional load balancing: hash(src_ip, dst_ip, src_port, dst_port, protocol)
  → Must parse to Layer 4
  → Fails for IPsec ESP (no visible ports)
  → Difficult with tunneled traffic

Flow Label load balancing: hash(src_ip, dst_ip, flow_label)
  → Only Layer 3 parsing needed
  → Works with all protocols
  → Flow Label persists across the path
  → Non-zero Flow Label implies consistent 5-tuple (RFC 6437)
```

## Linux ECMP with Flow Labels

```bash
# Check if Linux ECMP uses Flow Label for hashing
cat /proc/sys/net/ipv6/flowlabel_state_ranges

# Enable Flow Label-aware ECMP hashing in the kernel
sudo sysctl -w net.ipv6.flowlabel_state_ranges=1

# Add multiple equal-cost routes (ECMP)
sudo ip -6 route add 2001:db8:2::/48 \
    nexthop via 2001:db8:1::1 dev eth0 weight 1 \
    nexthop via 2001:db8:1::2 dev eth1 weight 1

# Verify ECMP routes
ip -6 route show 2001:db8:2::/48

# The kernel will distribute flows across both nexthops
# using (src, dst, flow_label) as the hash key
```

## HAProxy Load Balancer with IPv6 Flow Label Awareness

```
# haproxy.cfg - IPv6 frontend with flow-based persistence
frontend ipv6_frontend
    bind [::]:443 ssl crt /etc/ssl/server.pem
    mode tcp
    default_backend ipv6_servers

backend ipv6_servers
    mode tcp
    balance source    # Hash on source address (includes flow context)
    server server1 [2001:db8:server:1::1]:8080 check
    server server2 [2001:db8:server:1::2]:8080 check
    server server3 [2001:db8:server:1::3]:8080 check
```

## NGINX Upstream with IPv6

```nginx
# nginx.conf - IPv6 upstream load balancing
upstream backend_pool {
    ip_hash;   # Hash on client IP (persistent per client)

    server [2001:db8:backend:1::1]:8080;
    server [2001:db8:backend:1::2]:8080;
    server [2001:db8:backend:1::3]:8080;
}

server {
    listen [::]:80;
    listen [::]:443 ssl;

    location / {
        proxy_pass http://backend_pool;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

## Custom Flow Label Load Balancer (Python)

```python
import socket
import hashlib
import struct

class IPv6FlowLabelLoadBalancer:
    """
    Stateless load balancer using IPv6 Flow Label for backend selection.
    """

    def __init__(self, backends: list):
        """
        Args:
            backends: List of (ipv6_address, port) tuples
        """
        self.backends = backends

    def select_backend(self, src_addr: str, dst_addr: str, flow_label: int) -> tuple:
        """
        Select a backend server using flow label-based hashing.
        Same flow label always maps to the same backend (stateless).

        Args:
            src_addr:   Client's IPv6 address
            dst_addr:   Service's IPv6 address
            flow_label: IPv6 Flow Label from packet header

        Returns:
            (backend_address, backend_port) tuple
        """
        # Hash on (src, dst, flow_label) — same inputs → same output
        src_bytes = socket.inet_pton(socket.AF_INET6, src_addr)
        dst_bytes = socket.inet_pton(socket.AF_INET6, dst_addr)
        fl_bytes  = struct.pack("!I", flow_label & 0xFFFFF)

        hash_input = src_bytes + dst_bytes + fl_bytes
        hash_value = int(hashlib.sha256(hash_input).hexdigest(), 16)

        # Select backend using modulo
        backend_index = hash_value % len(self.backends)
        return self.backends[backend_index]

# Example usage
lb = IPv6FlowLabelLoadBalancer([
    ("2001:db8:backend::1", 8080),
    ("2001:db8:backend::2", 8080),
    ("2001:db8:backend::3", 8080),
])

# Simulate requests from the same client flow
client = "2001:db8:client::1"
service = "2001:db8:service::1"
flow = 0x2A3B4

# Same flow always goes to same backend
for _ in range(5):
    backend = lb.select_backend(client, service, flow)
    print(f"Flow 0x{flow:05X} → {backend}")
# All 5 should print the same backend
```

## Handling Zero Flow Labels

When Flow Label = 0, fall back to 5-tuple hashing:

```python
def select_backend_with_fallback(
    src_addr, dst_addr, flow_label,
    src_port=0, dst_port=0, protocol=6
):
    """Fall back to 5-tuple if flow label is zero."""
    if flow_label != 0:
        return select_by_flow_label(src_addr, dst_addr, flow_label)
    else:
        return select_by_5tuple(src_addr, dst_addr, src_port, dst_port, protocol)
```

## Conclusion

IPv6 Flow Labels enable truly stateless load balancing at Layer 3, without maintaining per-connection state tables. The same (src, dst, flow_label) triple always produces the same hash, ensuring all packets from a flow reach the same backend. This is especially valuable for encrypted traffic where port numbers are unavailable, and for high-performance environments where maintaining per-connection state is too expensive.
