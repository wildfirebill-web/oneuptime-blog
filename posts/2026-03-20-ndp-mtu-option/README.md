# How to Understand the MTU Option in NDP

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: NDP, MTU Option, Router Advertisement, IPv6, Path MTU

Description: Understand the NDP MTU option in Router Advertisements, how it propagates link MTU to hosts, and why it is important for links with non-standard MTUs.

## Introduction

The NDP MTU option (Type 5) in Router Advertisements allows routers to explicitly communicate the link MTU to all hosts on the segment. This is especially important on links where the MTU is not the standard 1500 bytes — such as jumbo-frame networks, PPPoE links, or tunnels — ensuring all hosts configure their interfaces with the correct MTU for PMTU discovery.

## MTU Option Format

```
NDP MTU Option (Type 5, Length = 1):

 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|     Type = 5  |    Length = 1 |           Reserved            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                              MTU                              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

Total: 8 bytes (Length=1 unit of 8 bytes)
MTU: 32-bit value, the link MTU in bytes

When to include:
  SHOULD be included on links where the MTU may vary
  MUST be consistent: all routers on the link MUST advertise the same MTU
  If not included: hosts use the interface MTU as configured
```

## Configuring MTU Option in radvd

```bash
# /etc/radvd.conf: include MTU option

# Standard Ethernet (1500 bytes) — include for clarity
sudo tee /etc/radvd.conf << 'EOF'
interface eth0 {
    AdvSendAdvert on;
    AdvLinkMTU 1500;    # MTU option: tell hosts MTU = 1500

    prefix 2001:db8::/64 {
        AdvOnLink on;
        AdvAutonomous on;
        AdvValidLifetime 2592000;
        AdvPreferredLifetime 604800;
    };
};
EOF

# Jumbo frames (9000 bytes)
sudo tee /etc/radvd.conf << 'EOF'
interface eth0 {
    AdvSendAdvert on;
    AdvLinkMTU 9000;    # MTU option: jumbo frames

    prefix 2001:db8::/64 {
        AdvOnLink on;
        AdvAutonomous on;
        AdvValidLifetime 2592000;
        AdvPreferredLifetime 604800;
    };
};
EOF

# PPPoE link (1492 bytes effective MTU)
sudo tee /etc/radvd.conf << 'EOF'
interface ppp0 {
    AdvSendAdvert on;
    AdvLinkMTU 1492;    # PPPoE: 1500 - 8 byte PPPoE header

    prefix 2001:db8::/64 {
        AdvOnLink on;
        AdvAutonomous on;
        AdvValidLifetime 2592000;
        AdvPreferredLifetime 604800;
    };
};
EOF

sudo systemctl restart radvd
```

## Verifying MTU Option Reception

```bash
# Check if hosts received the MTU from RA
# After receiving RA with MTU option, hosts should update their interface MTU

# Check the interface MTU (should match RA-advertised MTU)
ip link show eth0 | grep mtu

# Use rdisc6 to see advertised MTU
sudo rdisc6 eth0 | grep -i mtu
# Expected: "MTU: 1500 bytes" or your custom value

# Use tcpdump to see MTU in RA
sudo tcpdump -i eth0 -vv "icmp6 and ip6[40] == 134" | grep -i mtu

# Linux applies MTU from RA (if accept_ra_mtu = 1)
cat /proc/sys/net/ipv6/conf/eth0/accept_ra_mtu
# 0 = ignore MTU in RA
# 1 = apply MTU from RA (default on some distributions)

# Enable applying MTU from RA
sudo sysctl -w net.ipv6.conf.eth0.accept_ra_mtu=1
```

## MTU Option in Practice

```python
import struct

def parse_mtu_option(option_data: bytes) -> dict:
    """Parse NDP MTU Option (Type 5)."""
    if len(option_data) < 8:
        raise ValueError("MTU option requires 8 bytes")

    opt_type, opt_len = struct.unpack("!BB", option_data[:2])
    reserved = struct.unpack("!H", option_data[2:4])[0]
    mtu = struct.unpack("!I", option_data[4:8])[0]

    return {
        "type": opt_type,
        "length_units": opt_len,
        "mtu": mtu,
        "meets_ipv6_minimum": mtu >= 1280,
    }

def build_mtu_option(mtu: int) -> bytes:
    """Build NDP MTU Option (Type 5)."""
    if mtu < 1280:
        raise ValueError("MTU must be at least 1280 bytes for IPv6")
    return struct.pack("!BBHI", 5, 1, 0, mtu)

# Test
for mtu in [1280, 1492, 1500, 9000]:
    option = build_mtu_option(mtu)
    parsed = parse_mtu_option(option)
    print(f"MTU option: {parsed['mtu']} bytes ({len(option)} bytes total)")
```

## Conclusion

The NDP MTU option propagates the link MTU to all hosts via Router Advertisements, ensuring consistent PMTU Discovery behavior across the network segment. It is especially important on non-standard-MTU links: PPPoE (1492), jumbo frames (9000), or any tunnel where the effective MTU differs from 1500 bytes. All routers on a link must advertise the same MTU value to avoid conflicting configuration on hosts. Hosts receiving the MTU option should apply it to their interface MTU, improving PMTU Discovery accuracy from the start.
