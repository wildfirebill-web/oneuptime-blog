# How to Identify Network Bottlenecks Using Traceroute

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Traceroute, Networking, Bottlenecks, Latency, IPv4, Performance

Description: Use traceroute output patterns to identify where latency spikes occur along a network path and pinpoint the specific link causing performance degradation.

A network bottleneck appears as a sudden jump in latency between two consecutive traceroute hops. Understanding how to read this pattern lets you identify the exact link or router causing slow performance.

## The Bottleneck Pattern

```bash
traceroute -n 8.8.8.8

 1  192.168.1.1      1ms    ← local gateway: 1ms
 2  10.1.0.1         8ms    ← ISP edge: +7ms (WAN link)
 3  72.14.0.1      150ms    ← JUMP! +142ms
 4  142.250.0.1    151ms    ← subsequent hops only +1ms
 5  8.8.8.8        152ms

# The bottleneck is the link BETWEEN hop 2 and hop 3
# The link 10.1.0.1 → 72.14.0.1 has 142ms of latency
# All subsequent hops just inherit the added latency
```

## Interpreting Latency Deltas

```bash
#!/bin/bash
# analyze-trace.sh — Show latency added at each hop

HOST="$1"
PREV=0

traceroute -n -q 1 "$HOST" 2>/dev/null | grep -E '^\s+[0-9]' | while read line; do
    HOP=$(echo "$line" | awk '{print $1}')
    IP=$(echo "$line" | awk '{print $2}')
    RTT=$(echo "$line" | awk '{print $3}')

    if [[ "$RTT" =~ ^[0-9]+\.[0-9]+$ ]]; then
        DELTA=$(echo "$RTT - $PREV" | bc 2>/dev/null || echo "?")
        echo "Hop $HOP: $IP — $RTT ms (+${DELTA} ms)"
        PREV=$RTT
    fi
done
```

## Differentiating Real vs Artificial Latency

Not all latency spikes are real bottlenecks:

```bash
traceroute -n 8.8.8.8

 3  72.14.0.1      150ms    ← spike here
 4  8.8.8.8         12ms    ← drops back down

# Wait — the destination (hop 4) is FASTER than hop 3
# This means: hop 3's ICMP response was deprioritized
# The ACTUAL path delay is fine
# Hop 3 delays ICMP Time Exceeded but passes traffic quickly

# A REAL bottleneck shows latency that persists in all subsequent hops:
 3  72.14.0.1      150ms
 4  142.0.0.1      151ms   ← still high
 5  8.8.8.8        152ms   ← destination is also high
```

## High Latency vs Packet Loss Bottlenecks

```bash
# Congested link shows:
# - High and variable RTT (not just high but also inconsistent)
# - Packet loss at the congested hop

# Run multiple probes for better picture
traceroute -q 5 -n 8.8.8.8    # 5 probes per hop

# Output shows 5 RTT values per hop:
# 3  72.14.0.1  12ms 12ms 150ms 11ms 11ms
# The single 150ms spike = transient burst loss or CPU congestion
# All 150ms = sustained bottleneck
```

## Bottleneck on Your LAN

```bash
# Hop 1 latency should be < 5ms
# If hop 1 shows 50ms+ on Ethernet → local network problem:
# - Port mismatch (half-duplex vs full-duplex)
# - Faulty cable
# - Overloaded switch CPU

# Check interface errors
ip -s link show eth0
# Look for: RX errors, TX errors, dropped packets
# High numbers indicate physical layer problems
```

## Using MTR for Continuous Bottleneck Detection

```bash
# MTR gives ongoing statistics, better for intermittent bottlenecks
sudo mtr --report --report-cycles=30 -n 8.8.8.8

# Fields:
# HOST: IP address
# Loss%: Packet loss percentage
# Snt: Packets sent
# Last: Last RTT
# Avg: Average RTT
# Best: Minimum RTT
# Wrst: Maximum RTT (worst)
# StDev: Standard deviation (jitter)

# High StDev at a specific hop = that link has variable latency = congestion
```

Identifying bottlenecks requires looking at where RTT jumps AND persists — a spike that recovers in the next hop is a router measurement artifact, not a real bottleneck.
