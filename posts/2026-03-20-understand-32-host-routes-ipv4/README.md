# How to Understand /32 Host Routes in IPv4

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv4, Routing, /32, Host Routes, Networking, Loopback

Description: A /32 host route identifies exactly one IPv4 address with a 32-bit mask, used for loopback interfaces, anycast IPs, policy routing, and precise static route injection.

## What Is a /32 Route?

A /32 mask is 255.255.255.255 - all 32 bits are the network bits. This means the "network" contains exactly one address: itself. There is no host portion and no broadcast.

## Common Uses

| Use Case | Example |
|----------|---------|
| Loopback interfaces | `10.0.0.1/32` on loopback |
| Anycast services | `203.0.113.10/32` on lo, advertised via BGP |
| Static routes to specific hosts | `route add 8.8.8.8/32 via gw` |
| MPLS LSPs | Labeled host routes for LSP endpoints |
| Blackhole routing | Null0 /32 to drop specific IP |

## Adding /32 Routes on Linux

```bash
# Route a specific host through a specific gateway

sudo ip route add 8.8.8.8/32 via 192.168.1.1 dev eth0

# Add a /32 to loopback (anycast or virtual service IP)
sudo ip addr add 10.255.0.1/32 dev lo

# View /32 routes in the routing table
ip route show | grep /32

# Add a blackhole /32 (drop traffic to a specific host)
sudo ip route add blackhole 203.0.113.100/32
```

## /32 in BGP Filtering

ISPs and enterprises often filter /32 routes to prevent table pollution, or specifically accept /32s for anycast:

```bash
# FRRouting: deny /32s in BGP updates from customers
# (prefix-list to filter overly specific routes)
ip prefix-list NO-HOST-ROUTES deny 0.0.0.0/0 ge 32
ip prefix-list NO-HOST-ROUTES permit 0.0.0.0/0 le 30
```

## Checking if an Address is a /32

```python
import ipaddress

def is_host_route(cidr: str) -> bool:
    """Return True if the CIDR is a /32 host route."""
    return ipaddress.IPv4Network(cidr, strict=False).prefixlen == 32

print(is_host_route("192.168.1.1/32"))  # True
print(is_host_route("192.168.1.0/24"))  # False
```

## /32 vs /31 vs Host Address

```python
import ipaddress

for cidr in ["10.0.0.1/32", "10.0.0.0/31", "10.0.0.0/24"]:
    net = ipaddress.IPv4Network(cidr, strict=False)
    print(f"{cidr:20s}  addresses={net.num_addresses}  "
          f"usable={max(net.num_addresses - 2, net.num_addresses if net.prefixlen >= 31 else 0)}")
```

## Key Takeaways

- A /32 identifies exactly one IP address with no broadcast or network distinction.
- Common on loopback interfaces, anycast VIPs, and precise policy routing entries.
- BGP blackhole routes often use /32 to drop traffic to compromised or abusive hosts.
- In routing table management, keep /32s controlled - they add entries without summarizing.
