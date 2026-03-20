# How to Troubleshoot IPv6 Path MTU Discovery Failures

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Path MTU Discovery, Troubleshooting, ICMPv6, Black Hole

Description: Diagnose and fix IPv6 Path MTU Discovery failures, identify PMTU black holes, and restore connectivity when large packets are silently dropped.

## Introduction

PMTU black holes occur when ICMPv6 Packet Too Big messages are blocked somewhere in the path. The result is that small packets succeed (TCP SYN/ACK, DNS, short HTTP requests) while large transfers (HTTPS pages, file downloads, SSH sessions after authentication) hang or fail silently. This asymmetry is the signature symptom of a PMTU failure.

## Recognizing PMTU Failure Symptoms

```text
Classic PMTU black hole symptoms:

1. TCP connections establish successfully (SYN is small)
2. HTTPS page partially loads then hangs
3. SSH connects but becomes unresponsive after login
4. File downloads start at a few KB then stall
5. Large DNS responses (DNSSEC) fail; small ones succeed
6. Ping6 works; curl/wget hangs after sending HTTP request
7. Symmetric failure: only affects traffic in one direction

Key test: ping6 -s 1452 destination
  → Success: PMTU is fine for 1500-byte packets
  → "Message too long" or no reply: PMTU issue
```

## Diagnosing PMTU Issues

```bash
# Step 1: Verify the connection works with small packets

ping6 -s 8 2001:db8::server   # 8-byte payload (tiny packet)
# Should succeed

# Step 2: Test with full-size packet
ping6 -M do -s 1452 2001:db8::server  # 1500-byte packet
# If this fails while small packets work: PMTU black hole

# Step 3: Check if ICMPv6 PTB messages are being received
sudo tcpdump -i eth0 -n "icmp6 and ip6[40] == 2" -v
# Watch for: "packet too big" messages from intermediate routers

# Step 4: Check PMTU cache
ip -6 route show cache | grep mtu
# If empty: no PTB messages received (or PMTUD not working)

# Step 5: Tracepath to discover path MTUs along the route
tracepath6 2001:db8::server
# Shows MTU at each hop - identifies the bottleneck

# Step 6: Use mtr for continuous path analysis
mtr -6 --report 2001:db8::server
```

## Identifying Where ICMPv6 Is Being Blocked

```bash
# Check local firewall rules for ICMPv6
sudo ip6tables -L -v -n | grep -E "icmpv6|icmp6"

# Check if PTB messages are generated but blocked outbound
sudo ip6tables -L OUTPUT -v -n | grep -E "icmpv6|icmp6|DROP|REJECT"

# Capture on all interfaces to see if PTB arrives but gets dropped
sudo tcpdump -i any -n "icmp6 and ip6[40] == 2" 2>/dev/null

# Check nftables ruleset
sudo nft list ruleset | grep -A2 "icmpv6"

# Check firewalld (if used)
sudo firewall-cmd --list-rich-rules | grep icmpv6
```

## Fixing PMTU Black Holes

The primary fix is to ensure ICMPv6 PTB messages are allowed through all firewalls:

```bash
# Fix 1: Allow ICMPv6 Packet Too Big at all firewall stages
sudo ip6tables -I INPUT 1 -p icmpv6 --icmpv6-type packet-too-big -j ACCEPT
sudo ip6tables -I OUTPUT 1 -p icmpv6 --icmpv6-type packet-too-big -j ACCEPT
sudo ip6tables -I FORWARD 1 -p icmpv6 --icmpv6-type packet-too-big -j ACCEPT

# Fix 2: If fixing the firewall is not possible, clamp TCP MSS
# This prevents large TCP segments without relying on PMTUD
sudo ip6tables -t mangle -A FORWARD \
    -p tcp --tcp-flags SYN,RST SYN \
    -j TCPMSS --clamp-mss-to-pmtu

# Fix 3: Set a conservative MSS manually (for known tunnel overhead)
# Normal MSS: 1500-40-20 = 1440; tunnel may reduce to 1280-40-20 = 1220
sudo ip6tables -t mangle -A FORWARD \
    -p tcp --tcp-flags SYN,RST SYN \
    -j TCPMSS --set-mss 1220

# Fix 4: If on a VPN/tunnel, set the tunnel interface MTU appropriately
sudo ip link set tun0 mtu 1280
```

## Automated PMTU Failure Detection Script

```python
import subprocess
import sys

def test_pmtu(destination: str, interface: str = "eth0") -> dict:
    """
    Test for PMTU black holes to a destination.
    Returns assessment of PMTU health.
    """
    results = {}

    # Test small packet
    small = subprocess.run(
        ["ping6", "-c", "3", "-s", "8", "-q", destination],
        capture_output=True, text=True
    )
    results["small_packet_ok"] = small.returncode == 0

    # Test full-size packet (no fragmentation)
    large = subprocess.run(
        ["ping6", "-c", "3", "-M", "do", "-s", "1452", "-q", destination],
        capture_output=True, text=True
    )
    results["large_packet_ok"] = large.returncode == 0

    # Check PMTU cache
    route = subprocess.run(
        ["ip", "-6", "route", "get", destination],
        capture_output=True, text=True
    )
    results["pmtu_cached"] = "mtu" in route.stdout

    # Diagnose
    if results["small_packet_ok"] and not results["large_packet_ok"]:
        results["diagnosis"] = "PMTU BLACK HOLE DETECTED"
        results["recommendation"] = "Allow ICMPv6 type 2 through all firewalls, or enable MSS clamping"
    elif results["small_packet_ok"] and results["large_packet_ok"]:
        results["diagnosis"] = "PMTU appears healthy"
        results["recommendation"] = "No action required"
    else:
        results["diagnosis"] = "Destination unreachable"
        results["recommendation"] = "Check basic IPv6 connectivity first"

    return results

result = test_pmtu("2001:db8::1")
for key, value in result.items():
    print(f"{key}: {value}")
```

## Conclusion

IPv6 PMTU failures manifest as connections that work for small data but fail for large transfers. The root cause is almost always ICMPv6 Packet Too Big messages being blocked by a firewall. The fix is to allow ICMPv6 type 2 messages through all firewalls. When changing firewall rules is not feasible, TCP MSS clamping provides a workaround that prevents the problem at the TCP layer. Always use `tracepath6` as the first diagnostic tool - it reveals the MTU at each hop and immediately shows where the path bottleneck exists.
