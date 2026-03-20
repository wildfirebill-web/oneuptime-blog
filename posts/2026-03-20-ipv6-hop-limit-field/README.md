# How to Understand the IPv6 Hop Limit Field

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Hop Limit, TTL, Networking, Routing

Description: Understand the IPv6 Hop Limit field, how it prevents routing loops, its relationship to IPv4 TTL, and how to configure and troubleshoot it in practice.

## Introduction

The IPv6 Hop Limit is an 8-bit field in the IPv6 header that limits the number of routers a packet can traverse before being discarded. It serves the same purpose as IPv4's TTL (Time To Live) - preventing packets from looping indefinitely in case of routing loops. Despite the name change, the behavior is identical: each router decrements the field by 1 and discards the packet (sending ICMPv6 Time Exceeded) when it reaches zero.

## How Hop Limit Works

```text
Source sets Hop Limit = 64
  ↓ Router 1 decrements → 63
  ↓ Router 2 decrements → 62
  ↓ Router 3 decrements → 61
  ...
  ↓ Router 64 decrements → 0
  ↓ Router drops packet + sends ICMPv6 Time Exceeded
```

## Default Hop Limit Values

```text
Typical initial values:
  64  - Linux, macOS (most Unix systems)
  128 - Windows
  255 - Cisco IOS routers (typically)
  64  - Most network equipment

Minimum required by RFC 8200: routers must forward packets with HL ≥ 1
Maximum: 255 (fits in 8 bits)
Special: 255 is used by NDP to ensure on-link delivery
  (Router Advertisements always have HL=255)
```

## Configuring Hop Limit on Linux

```bash
# View current default Hop Limit

cat /proc/sys/net/ipv6/conf/all/hop_limit
# Output: 64

# Change the default Hop Limit
sudo sysctl -w net.ipv6.conf.all.hop_limit=128

# Per-interface setting
sudo sysctl -w net.ipv6.conf.eth0.hop_limit=64

# Make permanent
echo "net.ipv6.conf.all.hop_limit=64" | sudo tee -a /etc/sysctl.conf

# Check the hop limit of packets reaching a destination
traceroute6 -n 2001:4860:4860::8888
# Each line shows the router and remaining hop limit context
```

## Using Hop Limit for Network Scoping

A special use of Hop Limit is to restrict packet delivery to a specific scope:

```text
Hop Limit = 1:  Only the directly connected segment (link-scope)
              → Used for NDP, Router Advertisements, etc.
              → Routers must NOT forward HL=1 packets

Hop Limit = 255 in received NDP:
              → Verifies the sender is on-link (cannot be spoofed from remote)
              → Routers decrement HL, so an off-link NDP packet would arrive with HL < 255
              → Linux kernel enforces: RA/NS/NA/RS MUST have HL=255
```

```bash
# Verify NDP packets have HL=255
sudo tcpdump -i eth0 -vv icmp6 | grep -E "hlim|icmp6"
# Should show: hlim 255 for NDP messages
# A Router Advertisement with hlim < 255 is a security violation
```

## Setting Hop Limit in Applications

```python
import socket

def create_ipv6_socket_with_hop_limit(hop_limit: int = 64) -> socket.socket:
    """
    Create an IPv6 socket with a specific Hop Limit.

    Args:
        hop_limit: Hop limit value (1-255)
    """
    sock = socket.socket(socket.AF_INET6, socket.SOCK_DGRAM)

    # IPV6_UNICAST_HOPS sets hop limit for unicast packets
    sock.setsockopt(socket.IPPROTO_IPV6, socket.IPV6_UNICAST_HOPS, hop_limit)

    # For multicast packets, use IPV6_MULTICAST_HOPS
    # sock.setsockopt(socket.IPPROTO_IPV6, socket.IPV6_MULTICAST_HOPS, hop_limit)

    return sock

# TTL-scoped multicast
# Hop Limit = 1  → link-local only
# Hop Limit = 16 → site-local (organization)
# Hop Limit = 64 → regional
# Hop Limit = 128 → continental
# Hop Limit = 255 → global

# Create a "site-local" multicast sender
site_multicast_sock = create_ipv6_socket_with_hop_limit(16)
```

## ICMPv6 Time Exceeded

When a router receives a packet with Hop Limit = 1 (decrements to 0), it sends back an ICMPv6 Time Exceeded message:

```bash
# Simulate a Hop Limit too small
ping6 -t 3 2001:db8::1  # Sends 3 hops max

# traceroute6 uses progressively increasing HL to map the path
traceroute6 2001:db8::1
# Each hop returns ICMPv6 Time Exceeded, revealing the router address
```

## Conclusion

The IPv6 Hop Limit field is functionally identical to IPv4's TTL - it prevents routing loops and can be used to scope packet delivery. Key differences from TTL: the name emphasizes it is a hop count (not a timer), NDP requires HL=255 for security, and applications can control it per-socket using `IPV6_UNICAST_HOPS`. The most common default is 64, which is sufficient for any real-world path while keeping the value visible and meaningful in traceroute output.
