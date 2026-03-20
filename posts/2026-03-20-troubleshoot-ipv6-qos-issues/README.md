# How to Troubleshoot IPv6 QoS Issues

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, QoS, Troubleshooting, DSCP, Latency, Jitter, Network Performance

Description: Diagnose and resolve common IPv6 QoS problems including DSCP marking not being preserved, traffic landing in wrong classes, and insufficient bandwidth allocation.

---

IPv6 QoS issues typically manifest as voice/video quality degradation, unexpected traffic prioritization, or DSCP markings being cleared by intermediate network devices. Systematic troubleshooting identifies whether the problem is at the marking, enforcement, or path level.

## Step 1: Verify DSCP Marking is Applied

```bash
# Check if DSCP marking rules are active

sudo nft list table ip6 mangle  # nftables
# OR
sudo ip6tables -t mangle -L -v -n  # ip6tables

# Verify marking is actually applied to packets
# Capture IPv6 VoIP traffic and check Traffic Class
sudo tcpdump -i eth0 -nn ip6 and "udp port 5060" -v -c 10

# Look for: "class 0xa0" (CS5 = 40) or "class 0xb8" (EF = 46)

# If not marked, test manually
sudo nft add rule ip6 mangle prerouting \
  meta l4proto udp udp dport 5060 ip6 dscp set cs5

# Or with ip6tables
sudo ip6tables -t mangle -A OUTPUT \
  -p udp --dport 5060 -j DSCP --set-dscp-class CS5
```

## Step 2: Check if DSCP is Preserved Through Network

```bash
# Common issue: intermediate routers remark or clear DSCP

# On source host - verify mark is set
sudo tcpdump -i eth0 -nn ip6 and "udp port 5060" -v | grep "class"

# On destination host - verify mark arrives
sudo tcpdump -i eth0 -nn ip6 and "udp port 5060" -v | grep "class"

# If mark is cleared between source and destination:
# Check each router in the path for DSCP remarking policies

# Trace path and check at each hop
traceroute6 2001:db8::destination

# Test with known DSCP value
python3 -c "
import socket, struct
# Create IPv6 socket with Traffic Class (DSCP EF = 46)
sock = socket.socket(socket.AF_INET6, socket.SOCK_DGRAM)
sock.setsockopt(socket.IPPROTO_IPV6, socket.IPV6_TCLASS, 46 << 2)
print('Socket created with DSCP EF')
sock.close()
"
```

## Step 3: Diagnose Wrong Traffic Classification

```bash
# Check which class traffic is landing in
sudo tc -s class show dev eth0

# If VoIP is in the default class instead of VoIP class:
# Check filter rules
sudo tc filter show dev eth0

# Verify u32 filter matches
sudo tc filter show dev eth0 | grep -A5 "protocol ipv6"

# Debug filter matching with pkt_type
sudo tc filter show dev eth0 root

# Test filter with specific source/destination
# Send test UDP to VoIP port and watch counters
nc -6 -u 2001:db8::voip-server 5060 &
sleep 2 && kill %1

# Check if counters changed in the expected class
sudo tc -s class show dev eth0 | grep -A3 "1:10"
```

## Step 4: Check Bandwidth Allocation

```bash
# View current bandwidth usage per class
watch -n 1 'sudo tc -s class show dev eth0 | grep -E "class|Sent|rate"'

# Check if class is oversubscribed (drops)
sudo tc -s class show dev eth0 | grep "dropped"

# If drops occurring in VoIP class:
# Check ceil value - may be limiting VoIP burst
sudo tc class show dev eth0 | grep "1:10"
# Increase ceil: sudo tc class change dev eth0 classid 1:10 htb rate 20mbit ceil 30mbit

# Check if total allocation exceeds interface capacity
# Sum of all class rates should not exceed TOTAL_BW

# Adjust bandwidth if oversubscribed
sudo tc class change dev eth0 parent 1:1 classid 1:20 \
  htb rate 400mbit ceil 600mbit
```

## Step 5: Diagnose Latency/Jitter Problems

```bash
# Measure IPv6 latency to VoIP destination
ping6 -c 50 -i 0.1 2001:db8::voip-server

# Calculate jitter (mdev value)
ping6 -c 100 2001:db8::voip-server | tail -3

# Check if VoIP class queue is causing latency
# PFIFO with small limit = less queuing delay
sudo tc qdisc show dev eth0 | grep "1:10"

# If using fq_codel on VoIP class, check target/interval
sudo tc qdisc show dev eth0 | grep -A3 "parent 1:10"

# Replace with PFIFO for minimum latency (VoIP only)
sudo tc qdisc del dev eth0 parent 1:10 2>/dev/null
sudo tc qdisc add dev eth0 parent 1:10 pfifo limit 50
```

## Step 6: Check IPv6-Specific Issues

```bash
# Check for IPv6 fragmentation (causes jitter)
sudo tcpdump -i eth0 -nn "ip6[6] == 44"  # Fragment header

# Fix: Ensure application sends packets < MTU
# Check path MTU
tracepath6 2001:db8::voip-server

# Check NDP issues (high latency on first packet)
ip -6 neigh show | grep "2001:db8::voip"
# If INCOMPLETE, NDP resolution may be causing delay

# Pre-populate NDP cache
sudo ip -6 neigh add 2001:db8::voip-server \
  lladdr aa:bb:cc:dd:ee:ff dev eth0

# Verify ICMPv6 Packet Too Big is not filtered
# (required for PMTU discovery that helps avoid fragmentation)
sudo ip6tables -L INPUT | grep "icmpv6"
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type packet-too-big -j ACCEPT
```

## Creating a QoS Diagnostic Report

```bash
#!/bin/bash
# qos_diagnostic.sh - Generate QoS diagnostic report

echo "=== IPv6 QoS Diagnostic Report $(date) ==="
echo ""
echo "--- Interface: eth0 ---"
ip -6 addr show eth0
echo ""
echo "--- tc Classes ---"
sudo tc -s class show dev eth0
echo ""
echo "--- tc Filters ---"
sudo tc filter show dev eth0
echo ""
echo "--- ip6tables mangle ---"
sudo ip6tables -t mangle -L -n -v
echo ""
echo "--- DSCP sample (last 10 IPv6 packets) ---"
sudo timeout 10 tcpdump -i eth0 -nn ip6 -v -c 10 2>/dev/null | grep "class"
echo ""
echo "--- Latency to default gateway ---"
GW=$(ip -6 route show default | awk '{print $3}' | head -1)
ping6 -c 5 $GW 2>/dev/null | tail -2
```

IPv6 QoS troubleshooting most commonly reveals DSCP markings being cleared by carrier routers (a policy enforcement rather than technical issue), traffic not matching filter rules due to incorrect u32 offset calculations, or VoIP traffic landing in the wrong class because the DSCP value at offset 1 of the IPv6 header differs from what ip6tables marked earlier in the chain.
