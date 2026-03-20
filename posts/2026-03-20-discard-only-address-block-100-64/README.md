# How to Understand the Discard-Only Address Block (100::/64)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Discard-Only, 100::/64, RFC 6666, Blackhole, Networking

Description: Understand the IPv6 discard-only address block 100::/64 (RFC 6666), analogous to IPv4's 192.0.2.0/24, used for routing black holes and sink holes in network operations.

## Introduction

RFC 6666 allocates `100::/64` as the IPv6 "Discard-Only" address block. Packets sent to these addresses should be silently dropped. It is the IPv6 equivalent of IPv4's `192.0.2.0/24` discard range and is used for operational purposes like black-hole routing and traffic sinkholes.

## Use Cases

### 1. BGP Black-Hole Routing

```bash
# Announce 100::/64 routes to trigger discard at each router
# Routers with this route silently drop matching traffic

# Linux: add a black-hole route pointing to 100::/64
ip -6 route add blackhole 100::/64

# Verify
ip -6 route show 100::/64
# blackhole 100::/64 dev lo
```

### 2. Null Routing for DoS Mitigation

```bash
# When under a DDoS attack, null-route the victim's /128
# Traffic gets dropped before reaching the server

# Attacker is at 2001:db8:attack::/48
# Sink their traffic into 100::/64 on the edge router
ip -6 route add 100::1/128 blackhole

# BGP community trigger: advertise with "blackhole" community
# to upstream providers
```

### 3. Service Discovery Experiments

```bash
# Use 100::/64 addresses in test environments as unreachable destinations
# Useful for testing timeout behavior and fallback mechanisms

# Test your application's IPv6 timeout handling
curl --max-time 5 http://[100::1]/test
# Should timeout after 5 seconds
```

## Packet Behavior

```python
import ipaddress
import socket

def is_discard_only(addr: str) -> bool:
    """Check if an address is in the discard-only block."""
    try:
        a = ipaddress.IPv6Address(addr)
        return a in ipaddress.IPv6Network("100::/64")
    except ValueError:
        return False

# Test
print(is_discard_only("100::1"))     # True
print(is_discard_only("100::ffff"))  # True
print(is_discard_only("100::1:0"))   # False (outside /64)
print(is_discard_only("::1"))        # False
```

## Filtering at Network Boundaries

```bash
# Ensure 100::/64 does not appear as a legitimate source or destination
# in production traffic

# Block as source (spoofed discard-only source)
ip6tables -A INPUT -s 100::/64 -j DROP
ip6tables -A FORWARD -s 100::/64 -j DROP

# Block as destination from internal hosts
# (Internal hosts should not send to discard-only)
ip6tables -A OUTPUT -d 100::/64 -j LOG --log-prefix "DISCARD-ONLY: "
ip6tables -A OUTPUT -d 100::/64 -j DROP
```

## NGINX — Reject Traffic from Discard-Only Block

```nginx
# nginx.conf — deny access from discard-only block
# (Defensive measure in case of routing misconfigurations)
geo $ipv6_geo {
    default 1;
    100::/64 0;  # Discard-only block — deny
}

server {
    if ($ipv6_geo = 0) {
        return 403;
    }
}
```

## Comparison with IPv4 Equivalents

| Purpose | IPv4 | IPv6 |
|---|---|---|
| Documentation | 192.0.2.0/24, 198.51.100.0/24, 203.0.113.0/24 | 2001:db8::/32 |
| Discard/Sink | 192.0.2.0/24 (partially) | 100::/64 |
| Benchmarking | 198.18.0.0/15 | 2001:2::/48 |
| Loopback | 127.0.0.0/8 | ::1/128 |

## Conclusion

The `100::/64` discard-only block provides a well-known IPv6 destination for black-hole routing and network experiments. Network operators use it for DDoS sink-hole routing; developers use it to test timeout behavior. Ensure `100::/64` is filtered at network boundaries and never appears in legitimate traffic. Use OneUptime's network monitoring to detect unexpected routing changes to this block.
