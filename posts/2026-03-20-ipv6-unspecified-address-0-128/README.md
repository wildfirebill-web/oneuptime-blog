# How to Understand the IPv6 Unspecified Address (::/128)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Unspecified Address, ::, RFC 4291, Networking, Wildcard

Description: Understand the IPv6 unspecified address ::/128, its role as a source address during initialization, and how it differs from the wildcard bind address.

## Introduction

The IPv6 unspecified address `::` (or `::/128`) serves as a placeholder when a device has no assigned IPv6 address yet. It is used as a source address during stateless address autoconfiguration (SLAAC) and DHCPv6 before a real address is available.

## Key Properties

| Property | Value |
|---|---|
| Address | :: |
| Prefix | ::/128 |
| IPv4 equivalent | 0.0.0.0 |
| Can be source? | Yes (only during initialization) |
| Can be destination? | No |
| Forwardable | No |
| Globally reachable | No |

## Uses of the Unspecified Address

### 1. Neighbor Solicitation for DAD

During Duplicate Address Detection, the MN uses `::` as the source before the address is confirmed.

```bash
# Capture DAD Neighbor Solicitations (source = ::)
sudo tcpdump -i eth0 -n "icmp6 and ip6[8]=0"
# ip6[8]=0 means IPv6 hop limit = 255, used for NDP
# More specifically: src :: in DAD
sudo tcpdump -i eth0 -n "icmp6 and ip6 src ::"
```

### 2. DHCPv6 Initial Messages

Clients without an IPv6 address use `::` as the source in initial DHCPv6 Solicit messages.

```bash
# Capture initial DHCPv6 Solicit packets (source = ::)
sudo tcpdump -i eth0 -n "udp port 546 or udp port 547" -v | \
  grep ":: >"
```

### 3. Wildcard Bind Address

`::` as a socket bind address means "all IPv6 interfaces" (wildcard). This is NOT the same as the unspecified address semantically.

```python
import socket

# Wildcard bind — listens on ALL IPv6 interfaces
server = socket.socket(socket.AF_INET6, socket.SOCK_STREAM)
server.setsockopt(socket.IPPROTO_IPV6, socket.IPV6_V6ONLY, 0)

# Bind to :: = listen on all interfaces (wildcard)
server.bind(("::", 8080, 0, 0))
server.listen(5)
print("Listening on all IPv6 interfaces on port 8080")

# Note: when IPV6_V6ONLY = 0, "::" also accepts IPv4 connections
# via IPv4-mapped IPv6 addresses (::ffff:x.x.x.x)
```

### IPv6-Only vs Dual-Stack Wildcard Bind

```python
import socket

def create_server(port: int, ipv6_only: bool = True):
    """
    Create server with clear IPv6 binding semantics.
    """
    server = socket.socket(socket.AF_INET6, socket.SOCK_STREAM)
    server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)

    if ipv6_only:
        # IPv6 only — :: only matches IPv6 connections
        server.setsockopt(socket.IPPROTO_IPV6, socket.IPV6_V6ONLY, 1)
    else:
        # Dual-stack — :: also accepts IPv4 via IPv4-mapped
        server.setsockopt(socket.IPPROTO_IPV6, socket.IPV6_V6ONLY, 0)

    server.bind(("::", port, 0, 0))
    server.listen(128)
    return server
```

## The Unspecified Address in Routing

```bash
# ::/0 is the default IPv6 route (not the unspecified address)
# This is different from ::/128
ip -6 route show default
# default via fe80::1 dev eth0

# ::/128 itself is never a routable destination
# Attempts to route to :: will fail
ip -6 route get ::
# Error: Address not available
```

## Distinguishing :: from ::1 and ::/0

```python
import ipaddress

addr = ipaddress.IPv6Address("::")

print(f"Is unspecified: {addr.is_unspecified}")  # True
print(f"Is loopback:    {addr.is_loopback}")     # False
print(f"Is private:     {addr.is_private}")       # False

# Default route prefix
default_route = ipaddress.IPv6Network("::/0")
print(f":: is in ::/0: {addr in default_route}")  # True
print(f"All addresses are in ::/0: "
      f"{ipaddress.IPv6Address('2001:db8::1') in default_route}")  # True
```

## Application Validation

```python
def is_valid_peer_address(addr: str) -> bool:
    """
    Return False if the address is the unspecified address
    (unsuitable as a peer or destination address).
    """
    try:
        a = ipaddress.ip_address(addr)
        if a.is_unspecified:
            return False  # :: is not a valid peer
        if a.is_loopback:
            return False  # ::1 is loopback only
        return True
    except ValueError:
        return False
```

## Conclusion

The IPv6 unspecified address `::` plays a specific protocol role during address initialization. When used as a socket bind address (wildcard), it means "all interfaces" — a semantically different use. Applications must not use `::` as a destination or peer address. Use OneUptime's validation checks to detect services inadvertently binding to `::` in production when `::1` was intended.
