# How to Understand Extension Header Drop Rates in Production Networks

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Extension Headers, Network Measurement, Middleboxes, Troubleshooting

Description: Understand why IPv6 extension headers are frequently dropped in production networks, how to measure drop rates, and what this means for protocol design.

## Introduction

Extensive measurement research has revealed that IPv6 extension headers are dropped at surprisingly high rates by production internet infrastructure. This is a practical deployment problem that affects fragmentation, IPsec, and any new protocol that relies on extension headers. Understanding the causes and measuring drop rates helps network engineers make informed decisions about extension header use.

## Documented Drop Rates

Research papers and IETF working group documents have measured:

```
Extension Header Type     | Approximate Drop Rate (internet paths)
Fragment (NH=44)         | ~30-50% of paths (critical - breaks fragmentation)
Routing (NH=43)          | ~40-60% of paths
Hop-by-Hop (NH=0)        | ~50-70% of paths (critical - breaks MLD)
Authentication (NH=51)   | ~40-50% of paths
ESP (NH=50)              | ~10-30% of paths (less dropped than others)
Destination Opt (NH=60)  | ~30-50% of paths
```

Note: These rates vary significantly by measurement methodology, path, and time. Rates have improved as networks upgrade but remain a real concern.

## Why Extension Headers Are Dropped

```
1. Misconfigured firewalls:
   Many firewall default policies drop anything "unusual"
   Including packets with extension headers as a "precaution"

2. ACL/filter limitations in older hardware:
   Some hardware cannot parse beyond the first extension header
   Default policy: if can't parse, drop

3. Security concerns (valid):
   RH0 routing header was a legitimate security threat (deprecated)
   Some operators block all routing headers out of caution

4. Fragment filtering (invalid but common):
   Firewalls blocking all fragments (prevents fragment reassembly attacks)
   But this also blocks legitimate fragmented IPv6 traffic

5. Deep Packet Inspection failure:
   DPI systems that cannot parse extension headers
   May drop packets they can't classify

6. Performance optimization:
   Hardware fast paths may not support all extension headers
   Packets with unusual headers sent to slow path → buffer overflow → drop
```

## Testing Extension Header Reachability

```bash
# Test if a path drops Fragment Header packets
# Method: Compare reachability with and without fragmentation

# Test 1: Normal packet (no extension headers)
ping6 -c 5 -s 56 target.example.com  # Small packet, no fragmentation needed

# Test 2: Fragmented packet (includes Fragment Header)
ping6 -c 5 -s 1400 -M want target.example.com  # Force fragmentation

# Compare loss rates - if Test 2 has more loss, Fragment Headers are being dropped

# Test using traceroute to find where extension headers are dropped
traceroute6 -n target.example.com         # Normal path
traceroute6 -n -f 44 target.example.com   # With Fragment Header (if supported)
```

## Python: Extension Header Probe Tool

```python
import subprocess
import sys

def test_extension_header(target: str, nh_value: int, nh_name: str) -> dict:
    """
    Test if extension headers are blocked on the path to a target.
    Returns reachability information.

    Note: This is a conceptual implementation.
    Real testing requires Scapy or raw sockets.
    """
    result = {
        "target": target,
        "extension_header": nh_name,
        "nh_value": nh_value,
    }

    # Test 1: Normal ping6 (no extension headers)
    proc = subprocess.run(
        ["ping6", "-c", "3", "-W", "2", target],
        capture_output=True, text=True
    )
    normal_reachable = proc.returncode == 0
    result["normal_reachable"] = normal_reachable

    # Test 2: If Fragment Header test, use large ping
    if nh_value == 44:
        proc = subprocess.run(
            ["ping6", "-c", "3", "-W", "2", "-s", "1400", target],
            capture_output=True, text=True
        )
        ext_reachable = proc.returncode == 0
        result["with_extension_header"] = ext_reachable
        result["dropped"] = normal_reachable and not ext_reachable

    return result

# Test fragment header (the most critical to test)
result = test_extension_header("2001:4860:4860::8888", 44, "Fragment")
print(f"Target: {result['target']}")
print(f"Normal ping: {'OK' if result['normal_reachable'] else 'FAIL'}")
print(f"Fragment Header: {'DROPPED' if result.get('dropped') else 'PASSED'}")
```

## Operational Implications

```
For network operators deploying new protocols:
  → Cannot rely on extension headers for critical functionality
  → Design protocols to work WITHOUT extension headers when possible
  → Use extension headers only for optional enhancements

For firewall operators:
  → RFC 7045 requires allowing most extension headers
  → Review your drop policies - are you dropping legitimate traffic?
  → Test: are you accidentally blocking your own VPN (ESP) or fragmented traffic?

For application developers:
  → Avoid large UDP datagrams that require fragmentation
  → Use TCP instead of UDP for large data (TCP handles fragmentation at the app level)
  → If using IPsec, test paths for ESP passthrough

For IETF protocol designers:
  → The "extension header problem" has led to shift away from new extension headers
  → New proposals increasingly use UDP-based tunneling instead of new IPv6 headers
```

## Measuring Your Own Network

```bash
# Quick self-test: can your network forward Fragment Headers?
# Send a fragmented ICMPv6 to a known good target

# Check if fragments are reaching external hosts
sudo ip6tables -I INPUT -m frag --fragfirst -j LOG --log-prefix "FRAG-IN: "
ping6 -s 1400 -M want 2001:4860:4860::8888
grep "FRAG-IN" /var/log/kern.log | tail -5

# Check if your ISP blocks fragments
sudo tcpdump -i eth0 -vv "ip6[6] == 44" &
ping6 -s 1400 -M want 2001:4860:4860::8888
sleep 5
kill %1
```

## Conclusion

Extension header drop rates represent a real deployment challenge for IPv6. The fragment header is the most critical: a 30-50% drop rate means roughly one-third of internet paths will silently discard fragmented IPv6 packets, causing connection failures that are difficult to diagnose. Network operators should audit their own firewall policies against RFC 7045 guidelines and ensure they are not inadvertently dropping legitimate traffic. For protocol designers, this landscape argues for designing new protocols to work without relying on extension header delivery.
