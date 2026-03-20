# How to Troubleshoot IPv6 MTU and Path MTU Discovery Issues

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, MTU, PMTUD, Troubleshooting, Network Diagnostics, ICMPv6

Description: Diagnose and fix IPv6 MTU black holes and Path MTU Discovery failures that cause large packets to be silently dropped, resulting in TCP connections that stall after handshake.

## Introduction

IPv6 does not allow routers to fragment packets — only the sender can fragment. Path MTU Discovery (PMTUD) relies on ICMPv6 "Packet Too Big" messages to inform senders of the maximum transmission unit along a path. When "Packet Too Big" messages are blocked by firewalls, large packets are silently dropped, causing TCP connections to complete the handshake but stall when sending data. This is called an "MTU black hole."

## Understanding IPv6 MTU

```
Minimum IPv6 MTU: 1280 bytes (all routers must support)
Default Ethernet MTU: 1500 bytes
IPv6 header: 40 bytes (vs IPv4's 20 bytes)
Effective payload: 1460 bytes for IPv6 TCP (without options)

Typical MTU issues:
- VPN tunnels: reduce effective MTU (e.g., 1280-1400)
- PPPoE: MTU 1492 (reduces to ~1452 for IPv6)
- 6in4 tunnels: MTU 1480 (reduces to ~1440 for IPv6 TCP)
```

## Step 1: Check Interface MTU

```bash
# Show MTU for all interfaces
ip link show | grep mtu

# Show specific interface MTU
ip link show dev eth0 | grep mtu

# Show IPv6-specific MTU settings
cat /proc/sys/net/ipv6/conf/eth0/mtu

# Change MTU
sudo ip link set dev eth0 mtu 1500
```

## Step 2: Test for MTU Black Holes

```bash
# Send large packets to test PMTUD
# IPv6 doesn't support DF bit (it's always "don't fragment")
# Large ping with specific size:
ping6 -c 3 -s 1400 2001:db8::1

# Test with maximum payload
ping6 -c 3 -s 1452 2001:db8::1

# If large pings fail but small ones succeed:
# PMTUD might be broken (Packet Too Big messages blocked)

# Progressive ping size to find MTU black hole:
for size in 1200 1300 1400 1440 1460 1480; do
    result=$(ping6 -c 1 -s $size -W 2 2001:db8::1 2>&1)
    if echo "$result" | grep -q "1 received"; then
        echo "Size $size: OK"
    else
        echo "Size $size: FAILED"
    fi
done
```

## Step 3: Check for PMTUD (Packet Too Big Messages)

```bash
# Capture ICMPv6 Packet Too Big messages (type 2)
sudo tcpdump -i eth0 -v "ip6 proto 58 and ip6[40] == 2"

# Check if Packet Too Big is being received
sudo tcpdump -i eth0 -v "icmp6 and ip6[40] == 2" 2>/dev/null

# Allow ICMPv6 Packet Too Big in firewall
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type 2 -j ACCEPT
sudo ip6tables -A FORWARD -p icmpv6 --icmpv6-type 2 -j ACCEPT
```

## Step 4: Find the Path MTU

```bash
# Use tracepath6 to discover path MTU
tracepath6 2001:db8::1

# tracepath6 output shows MTU at each hop:
# 1:  2001:db8::100                              0.234ms
# 1:  2001:db8::1                                0.456ms
#     Route MTU: 1500

# traceroute6 with PMTUD information
traceroute6 -n 2001:db8::1

# Check kernel's cached path MTU
ip -6 route show cache | grep mtu
```

## Step 5: Configure TCP MSS Clamping (Workaround)

When PMTUD can't be fixed end-to-end, clamp TCP MSS at the router:

```bash
# Clamp TCP MSS for IPv6 traffic (on router/gateway)
sudo ip6tables -t mangle -A FORWARD \
    -p tcp --tcp-flags SYN,RST SYN \
    -j TCPMSS --clamp-mss-to-pmtu

# Or set specific MSS value (MTU - 60 bytes for IPv6+TCP headers)
# For 1280 MTU path: MSS = 1280 - 40 (IPv6) - 20 (TCP) = 1220
sudo ip6tables -t mangle -A FORWARD \
    -p tcp --tcp-flags SYN,RST SYN \
    -j TCPMSS --set-mss 1220
```

## Step 6: Set Explicit Interface MTU for VPN/Tunnel

```bash
# For 6in4 tunnels, reduce MTU to account for tunnel overhead
sudo ip link set dev sit0 mtu 1480

# For OpenVPN over IPv6
# In OpenVPN config:
# tun-mtu 1400
# fragment 1300
# mssfix 1300

# For WireGuard IPv6
# Interface MTU should be 1280 minimum
# Table 0 prevents routing loops
```

## Diagnostic Script

```bash
#!/bin/bash
# diagnose-ipv6-mtu.sh

TARGET="${1:-2001:4860:4860::8888}"
echo "=== IPv6 MTU Diagnostics ==="

echo ""
echo "Interface MTUs:"
ip link show | grep "mtu" | awk '{print "  "$2, "mtu:", $5}'

echo ""
echo "Path MTU discovery to $TARGET:"
tracepath6 "$TARGET" 2>/dev/null | grep -E "pmtu|MTU|mtu" | head -5

echo ""
echo "Packet size test to $TARGET:"
for size in 576 1000 1280 1400 1452; do
    if ping6 -c 1 -s "$size" -W 3 "$TARGET" &>/dev/null; then
        echo "  $size bytes: OK"
    else
        echo "  $size bytes: FAILED"
    fi
done
```

## Conclusion

IPv6 MTU issues typically manifest as connections that complete the TCP handshake but stall when transferring data. The cause is almost always a firewall blocking ICMPv6 "Packet Too Big" messages (type 2), preventing PMTUD from working. Fix by allowing ICMPv6 type 2 through all firewalls on the path. If PMTUD can't be fixed end-to-end, clamp TCP MSS with `ip6tables -t mangle` to prevent packets exceeding the path MTU from being sent.
