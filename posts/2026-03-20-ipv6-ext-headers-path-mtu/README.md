# How to Understand the Impact of Extension Headers on Path MTU Discovery

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Extension Headers, Path MTU, MTU Discovery, Networking

Description: Understand how IPv6 extension headers reduce the available payload space and interact with Path MTU Discovery, causing connectivity issues when not accounted for.

## Introduction

IPv6 extension headers consume bytes from the available payload space, effectively reducing the usable MTU for application data. When extension headers are present, Path MTU Discovery (PMTUD) must account for this overhead, or fragmentation and connectivity failures can occur. This interaction is especially important for tunneled traffic, IPsec, and any protocol that adds extension headers to packets.

## How Extension Headers Reduce Effective MTU

```text
Standard IPv6 packet (no extension headers, 1500 MTU):
  [IPv6 (40)] [TCP (20)] [App Data (1440)]
  Available application data: 1440 bytes

With IPsec ESP:
  [IPv6 (40)] [ESP (8)] [TCP (20)] [App Data] [ESP Trailer (12)] [ICV (16)]
  Available application data: 1500 - 40 - 8 - 20 - 12 - 16 = 1404 bytes

With Fragment Header + IPsec ESP (tunnel mode):
  [IPv6 (40)] [Fragment (8)] [ESP (8)] [Original IPv6 (40)] [TCP (20)] [App Data] [ICV (16)]
  Available application data: 1500 - 40 - 8 - 8 - 40 - 20 - 16 = 1368 bytes
```

## Calculating Effective MTU with Extension Headers

```python
def calculate_effective_mtu(
    link_mtu: int = 1500,
    extension_headers: dict = None
) -> dict:
    """
    Calculate available application data given link MTU and extension headers.

    Args:
        link_mtu: Physical link MTU in bytes
        extension_headers: Dict of {name: bytes} for each extension header

    Returns:
        Dictionary with various payload sizes
    """
    if extension_headers is None:
        extension_headers = {}

    IPV6_BASE = 40
    TCP_HEADER = 20
    UDP_HEADER = 8

    ext_header_total = sum(extension_headers.values())
    total_overhead_tcp = IPV6_BASE + ext_header_total + TCP_HEADER
    total_overhead_udp = IPV6_BASE + ext_header_total + UDP_HEADER

    return {
        "link_mtu": link_mtu,
        "ipv6_header": IPV6_BASE,
        "extension_headers": extension_headers,
        "extension_header_bytes": ext_header_total,
        "tcp_mss": link_mtu - total_overhead_tcp,
        "udp_payload": link_mtu - total_overhead_udp,
        "minimum_packet_size": IPV6_BASE + ext_header_total,
        "meets_min_mtu": link_mtu >= 1280,
    }

# Common scenarios

print("=== No Extension Headers ===")
result = calculate_effective_mtu(1500)
print(f"TCP MSS: {result['tcp_mss']} bytes")
print(f"UDP max: {result['udp_payload']} bytes")

print("\n=== IPsec ESP (Transport Mode) ===")
result = calculate_effective_mtu(1500, {
    "ESP header": 8,
    "ESP trailer (padding + next header)": 2,
    "ESP ICV (AES-GCM-128)": 16,
})
print(f"TCP MSS: {result['tcp_mss']} bytes")

print("\n=== IPv6-in-IPv6 Tunnel + ESP (Tunnel Mode) ===")
result = calculate_effective_mtu(1500, {
    "Fragment Header": 8,
    "ESP header": 8,
    "Inner IPv6 header": 40,
    "ESP ICV": 16,
})
print(f"TCP MSS: {result['tcp_mss']} bytes")

print("\n=== GRE Tunnel over IPv6 ===")
result = calculate_effective_mtu(1500, {
    "GRE header": 4,
})
print(f"TCP MSS: {result['tcp_mss']} bytes")
```

## Path MTU Discovery with Extension Headers

RFC 8201 defines IPv6 Path MTU Discovery. When extension headers are present, the source must account for their overhead when sending:

```bash
# Check current PMTU cache
ip -6 route show cache | grep mtu

# Flush the PMTU cache (forces rediscovery)
sudo ip -6 route flush cache

# Watch for ICMPv6 Packet Too Big messages (indicates PMTUD in action)
sudo tcpdump -i eth0 "icmp6 and ip6[40] == 2" &

# Check if PMTU discovery is enabled
cat /proc/sys/net/ipv6/conf/all/path_mtu_discovery
# 1 = enabled (default)
```

## MSS Clamping for Extension Headers

When packets traverse links with reduced MTU due to extension headers, MSS clamping ensures TCP doesn't exceed the path capacity:

```bash
# Clamp TCP MSS for IPv6 to account for IPsec overhead
# Normal MSS: 1500 - 40 (IPv6) - 20 (TCP) = 1440
# With IPsec ESP: 1500 - 40 - 20 (ESP) - 20 (TCP) = 1420

sudo ip6tables -t mangle -A FORWARD \
    -p tcp --tcp-flags SYN,RST SYN \
    -j TCPMSS --set-mss 1350  # Conservative value for tunneled connections

# Or use clamp-to-pmtu (dynamic, preferred)
sudo ip6tables -t mangle -A FORWARD \
    -p tcp --tcp-flags SYN,RST SYN \
    -j TCPMSS --clamp-mss-to-pmtu
```

## Extension Header Minimum MTU Requirements

IPv6 requires that every link have a minimum MTU of 1280 bytes. Extension headers do not change this requirement, but they do reduce the available application data:

```text
Minimum MTU: 1280 bytes
IPv6 base header: 40 bytes
Minimum application payload space: 1280 - 40 = 1240 bytes

With mandatory Hop-by-Hop header for jumbograms: 8 bytes
Minimum application payload space: 1280 - 40 - 8 = 1232 bytes

With Fragment Header + Hop-by-Hop:
  1280 - 40 - 8 - 8 = 1224 bytes minimum payload
```

## Conclusion

Extension headers reduce the effective available payload space proportionally to their size. This is most critical in tunneling scenarios (IPv6-in-IPv6, GRE, IPsec tunnel mode) where multiple headers stack up. Path MTU Discovery handles this correctly in theory, but relies on ICMPv6 Packet Too Big messages being delivered - which is not always the case due to firewall misconfiguration. Always configure MSS clamping on tunnel endpoints and interfaces where MTU is constrained by extension header overhead.
