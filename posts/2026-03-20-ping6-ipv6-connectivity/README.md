# How to Use ping6 for IPv6 Connectivity Testing

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, ping6, Connectivity Testing, Network Diagnostics, Linux, Troubleshooting

Description: Use ping6 (and ping -6) to test IPv6 connectivity, diagnose network reachability, and interpret ICMPv6 responses for IPv6 troubleshooting.

## Introduction

`ping6` (or `ping -6` on modern systems) sends ICMPv6 Echo Request packets to test IPv6 connectivity. It is the first tool to reach for when diagnosing IPv6 network issues, validating that a host is reachable, and measuring round-trip latency over IPv6.

## Basic Usage

```bash
# Ping an IPv6 address
ping6 2001:db8::1

# On modern Linux, use ping with -6 flag
ping -6 2001:db8::1

# Ping by hostname (uses AAAA record)
ping6 ipv6.google.com

# Ping IPv6 loopback
ping6 ::1
```

## Ping Options

```bash
# Send a specific number of pings
ping6 -c 4 2001:db8::1

# Set packet size (data payload)
ping6 -s 1200 2001:db8::1

# Set hop limit (TTL equivalent)
ping6 -t 5 2001:db8::1

# Set interval between pings (seconds)
ping6 -i 0.5 2001:db8::1

# Flood ping (requires root) - for performance testing
sudo ping6 -f -c 1000 2001:db8::1

# Verbose output with timing
ping6 -v 2001:db8::1
```

## Pinging Link-Local Addresses

Link-local addresses require specifying the interface:

```bash
# Using % notation for scope ID
ping6 fe80::1%eth0

# Using -I flag to specify interface
ping6 -I eth0 fe80::1

# List your link-local addresses first
ip -6 addr show scope link | grep inet6

# Ping a neighbor's link-local address
# Find neighbors with Neighbor Discovery
ip -6 neigh show | grep "REACHABLE"
```

## Interpreting ping6 Output

```bash
# Successful output
PING ipv6.google.com(2607:f8b0:4004:c1b::65e (2607:f8b0:4004:c1b::65e)) 56 data bytes
64 bytes from 2607:f8b0:4004:c1b::65e: icmp_seq=1 ttl=119 time=12.4 ms
64 bytes from 2607:f8b0:4004:c1b::65e: icmp_seq=2 ttl=119 time=11.8 ms

--- ipv6.google.com ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 11.8/12.1/12.4/0.3 ms
```

Key fields:
- `ttl`: Hop limit remaining (starts at 64 or 128, decremented at each hop)
- `time`: Round-trip time in milliseconds
- `icmp_seq`: Sequence number (gaps indicate lost packets)

## Common Error Messages and Causes

```bash
# "Network is unreachable"
ping6 2001:db8::1
# PING 2001:db8::1(2001:db8::1) 56 data bytes
# connect: Network is unreachable
# → No IPv6 default route. Check: ip -6 route show default

# "Destination unreachable: Address unreachable"
# → The path exists but the host is not responding

# "Destination unreachable: No route to host"
# → No IPv6 route to the destination

# "Unknown host"
ping6 ipv6.example.com
# → No AAAA record found in DNS

# No response (timeout)
# → Firewall blocking ICMPv6, or host is down
```

## Diagnosing IPv6 Connectivity Issues

```bash
#!/bin/bash
# ipv6-connectivity-check.sh

echo "=== IPv6 Connectivity Diagnostics ==="

# 1. Check IPv6 is configured
echo -n "IPv6 configured: "
ip -6 addr show scope global | grep -q "inet6" && echo "YES" || echo "NO"

# 2. Check default route
echo -n "Default IPv6 route: "
ip -6 route show default | head -1 || echo "NONE"

# 3. Ping loopback
echo -n "IPv6 loopback: "
ping6 -c 1 -W 1 ::1 &>/dev/null && echo "OK" || echo "FAIL"

# 4. Ping link-local gateway
GATEWAY=$(ip -6 route show default | grep -oE 'via [0-9a-f:]+' | awk '{print $2}')
if [ -n "$GATEWAY" ]; then
    IFACE=$(ip -6 route show default | grep -oE 'dev \w+' | awk '{print $2}')
    echo -n "Gateway $GATEWAY: "
    ping6 -c 1 -W 2 "$GATEWAY%$IFACE" &>/dev/null && echo "OK" || echo "FAIL"
fi

# 5. Ping internet IPv6
echo -n "Google IPv6 (2001:4860:4860::8888): "
ping6 -c 1 -W 3 2001:4860:4860::8888 &>/dev/null && echo "OK" || echo "FAIL"

# 6. DNS over IPv6
echo -n "IPv6 DNS resolution: "
ping6 -c 1 -W 3 ipv6.google.com &>/dev/null && echo "OK" || echo "FAIL"
```

## Using ping6 to Test Path MTU

```bash
# Test with different packet sizes to find MTU
for size in 1400 1450 1480 1492 1500; do
    result=$(ping6 -c 1 -s $size -M do 2001:db8::1 2>&1)
    if echo "$result" | grep -q "1 received"; then
        echo "Size $size: OK"
    else
        echo "Size $size: FRAGMENTED or LOST"
    fi
done
```

## Conclusion

`ping6` (or `ping -6`) is the foundation of IPv6 connectivity testing. Use `-c` for fixed packet counts, `-I interface` for link-local pings, and interpret the `ttl` field to estimate the number of hops to the destination. When ping6 fails with "Network is unreachable," check the IPv6 default route with `ip -6 route show default` before investigating further.
