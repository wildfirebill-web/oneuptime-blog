# How to Troubleshoot TCP Out-of-Order Packets

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: TCP, Out-of-Order, ECMP, Networking, Troubleshooting, Wireshark

Description: Diagnose TCP out-of-order packet delivery caused by ECMP routing, asymmetric paths, or network reordering, and apply fixes to reduce performance impact.

## Introduction

TCP packets can arrive out of sequence when they take different paths through the network - common in ECMP (Equal-Cost Multi-Path) environments, load-balanced networks, and wireless links. While TCP handles reordering correctly, excessive reordering triggers false duplicate ACKs and unnecessary retransmissions, degrading performance.

## Detecting Out-of-Order Packets

```bash
# Kernel counter for out-of-order segments

nstat | grep OutOfOrder
# TcpExtTCPOFOQueue: packets queued in out-of-order queue
# TcpExtTCPOFODrop: out-of-order packets dropped (OFO queue full)
# TcpExtTCPOFOMerge: OFO packets merged with main receive buffer

# Watch in real time
watch -n 1 "nstat -z | grep OFO"

# Also check tcpretransmit stats - spurious retransmits from reordering
nstat | grep Spurious
```

## Capturing Out-of-Order Packets

```bash
# Capture and identify OOO delivery
tcpdump -i eth0 -n -w /tmp/ooo.pcap 'tcp and host 10.20.0.5'

# Analyze with Wireshark filter
# In Wireshark: tcp.analysis.out_of_order

# Count OOO events
tshark -r /tmp/ooo.pcap -Y "tcp.analysis.out_of_order" 2>/dev/null | wc -l
```

## Causes of Out-of-Order Packets

### Cause 1: ECMP Routing

```bash
# ECMP sends packets over different paths with different latencies
# Some paths arrive later, causing reordering

# Verify ECMP is active
ip route show | grep "nexthop"
# If multiple nexthops: ECMP is active

# Check if different paths have different latencies
# Ping via each path using source routing
ping -c 5 -I 192.168.1.10 10.20.0.5   # via first gateway
ping -c 5 -I 192.168.2.10 10.20.0.5   # via second gateway
# If RTTs differ significantly: ECMP causes reordering
```

### Cause 2: Asymmetric Routing

```bash
# Check if forward and return paths differ
traceroute -n 10.20.0.5              # Forward path
# From remote: traceroute -n <your-ip>  # Return path
# Different hop counts = asymmetric paths
```

## Tuning Reordering Tolerance

```bash
# Increase TCP's reordering tolerance before triggering fast retransmit
sysctl net.ipv4.tcp_reordering
# Default: 3 (triggers fast retransmit on 3 dup ACKs)

# For high-reordering networks, increase tolerance
sysctl -w net.ipv4.tcp_reordering=6   # Tolerate more OOO before assuming loss

# Persist
echo "net.ipv4.tcp_reordering=6" >> /etc/sysctl.conf
```

## Fixing ECMP-Induced Reordering

```bash
# Fix 1: Ensure ECMP hashes on L4 (flow-based) not just L3 (per-packet)
# Linux ECMP hash policy
sysctl net.ipv4.fib_multipath_hash_policy
# 0 = L3 only (can reorder within a flow!)
# 1 = L3 + L4 (flow-based, no reordering within a flow)

# Set to flow-based to prevent per-packet load balancing
sysctl -w net.ipv4.fib_multipath_hash_policy=1

# Fix 2: Use Paris traceroute to verify flow consistency
apt install paris-traceroute
paris-traceroute 10.20.0.5
# Paris traceroute keeps the flow hash consistent across probes
# Consistent hops = flow-based ECMP (good)
# Varying hops = per-packet ECMP (bad for TCP)
```

## Measuring the Impact

```bash
# Before fix: measure spurious retransmits
nstat -z | grep Spurious
iperf3 -c 10.20.0.5 -t 30 | grep Retr

# Apply fix (fib_multipath_hash_policy=1)
sysctl -w net.ipv4.fib_multipath_hash_policy=1

# After fix
nstat -z | grep Spurious   # Should be lower
iperf3 -c 10.20.0.5 -t 30 | grep Retr  # Retransmits should drop
```

## Conclusion

TCP out-of-order packets typically originate from ECMP per-packet load balancing or asymmetric routing. Setting `fib_multipath_hash_policy=1` ensures ECMP load balances by flow (5-tuple) rather than per-packet, eliminating within-flow reordering. For residual reordering, increasing `tcp_reordering` tolerance prevents false fast-retransmit triggers. Monitor the OFO and Spurious retransmit kernel counters to quantify the problem and confirm the fix.
