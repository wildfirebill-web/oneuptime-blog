# How to Use /31 Subnets on Point-to-Point Links (RFC 3021)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv4, RFC 3021, Subnetting, Point-to-Point, Networking, WAN

Description: RFC 3021 defines the use of /31 subnets on point-to-point links, allowing both addresses to be usable host addresses with no network or broadcast address, saving IP space compared to /30.

## What RFC 3021 Defines

A /31 subnet has only 2 addresses. Normally, a 2-address subnet wastes both on network and broadcast, leaving 0 usable hosts. RFC 3021 (2000) specifies that on point-to-point links, both addresses can be used as host addresses — there is no broadcast on a P2P link.

## /31 vs /30 Comparison

| Property | /30 | /31 |
|----------|-----|-----|
| Total addresses | 4 | 2 |
| Usable hosts | 2 | 2 |
| Network address | .0 | Both are host addresses |
| Broadcast address | .3 | None |
| Address savings per link | — | 2 addresses |

With 1000 P2P links, /31 saves 2000 addresses compared to /30.

## Python: Generating /31 Pairs

```python
import ipaddress

def p2p_links_31(parent: str) -> list:
    """Generate /31 P2P link pairs from a parent block."""
    net = ipaddress.IPv4Network(parent, strict=False)
    links = []
    for subnet in net.subnets(new_prefix=31):
        addrs = list(subnet)
        links.append((str(addrs[0]), str(addrs[1])))
    return links

# Generate P2P links from 10.254.0.0/24
links = p2p_links_31("10.254.0.0/24")
print(f"Total /31 P2P links: {len(links)}")   # 128
for r1, r2 in links[:5]:
    print(f"  {r1} <-> {r2}")
```

Output:
```
Total /31 P2P links: 128
  10.254.0.0 <-> 10.254.0.1
  10.254.0.2 <-> 10.254.0.3
  10.254.0.4 <-> 10.254.0.5
  ...
```

## Configuring a /31 Link on Linux

```bash
# Side A
sudo ip addr add 10.254.0.0/31 dev eth1

# Side B
sudo ip addr add 10.254.0.1/31 dev eth1

# Ping the other side
ping 10.254.0.1  # From side A
```

## Configuring on Cisco IOS

```
! Router A
interface Serial0/0
  ip address 10.254.0.0 255.255.255.254

! Router B
interface Serial0/0
  ip address 10.254.0.1 255.255.255.254
```

## Compatibility Notes

- Linux: fully supports /31 natively.
- Cisco IOS 12.2+: supports /31 with `no ip directed-broadcast`.
- Some older equipment may not support /31; use /30 for maximum compatibility.
- Verify with `ping` after configuration; if ping fails, fall back to /30.

## Key Takeaways

- /31 provides 2 usable host addresses with no broadcast — perfect for P2P links.
- Saves 2 IP addresses per link compared to /30 (significant at scale).
- Supported on modern Cisco IOS, Linux, and most modern network equipment.
- Always test /31 connectivity before deploying in production environments.
