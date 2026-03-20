# How to Diagnose TCP Selective Acknowledgment (SACK) Problems

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: TCP, SACK, Selective Acknowledgment, Performance, Debugging, Linux

Description: Diagnose TCP SACK negotiation failures, SACK scoreboard issues, and performance problems caused by disabled or mishandled selective acknowledgments.

## Introduction

TCP Selective Acknowledgment (SACK) allows a receiver to acknowledge non-contiguous blocks of received data. Without SACK, the sender must retransmit everything from the first lost packet forward. With SACK, only the missing segments are retransmitted. This makes a significant difference on lossy links. SACK problems arise when one side doesn't support it, when middleboxes strip SACK options, or when the SACK scoreboard becomes inconsistent.

## Verifying SACK is Enabled

```bash
# Check Linux SACK settings
sysctl net.ipv4.tcp_sack
# 1 = enabled (default), 0 = disabled

# Check SACK FACK (Forward ACK) - additional SACK optimization
sysctl net.ipv4.tcp_fack
# Deprecated in kernel 4.15+, replaced by improved SACK handling

# Check DSACK (Duplicate SACK) - detect spurious retransmissions
sysctl net.ipv4.tcp_dsack
# 1 = enabled (default)
```

## Check if SACK is Being Negotiated

```bash
# Capture a connection handshake and check for SACK option in SYN
tcpdump -i eth0 -n 'tcp[tcpflags] & tcp-syn != 0' -v 2>/dev/null | head -30
# Look for: options [mss 1460,sackOK,TS val ...

# In Wireshark:
# Filter: tcp.flags.syn == 1
# Look at "TCP Options" in packet details
# Should show: SACK permitted option (kind=4, len=2)

# If SYN has sackOK but SYN-ACK doesn't: remote disabled SACK
# Both must advertise sackOK in SYN/SYN-ACK for SACK to be used
```

## Check SACK Statistics

```bash
# SACK-related kernel counters
nstat -a | grep -i sack

# Key counters:
# TcpExtTCPSACKReneging  → Receiver "reneged" SACK (said received, now says missing)
#                          Indicates buggy middlebox or broken implementation
# TcpExtTCPSACKReorder   → Reordering detected via SACK blocks
# TcpExtTCPSACKDiscard   → SACK blocks discarded (out of order, overlap)
# TcpExtTCPSackFailures  → Sender failed to retransmit per SACK hint
# TcpExtTCPSACKShifted   → SACK coalescing happened
# TcpExtTCPDSACKOfoRecv  → DSACK for out-of-order received (spurious retransmit detection)
# TcpExtTCPDSACKRecv     → DSACK received (we sent something spuriously)

# Monitor over time during a transfer:
watch -n 2 'nstat -z | grep -i sack'
```

## SACK Reneging Problem

```bash
# SACK reneging: receiver told us it has data, then claims it doesn't
# This causes sender to retransmit data receiver already has

# Symptoms:
# TcpExtTCPSACKReneging counter increasing
# Performance degradation on high-BDP links
# Wireshark shows ACK going backwards

# Wireshark detection:
# Statistics → Expert Information → look for "SACK reneging"

# Causes:
# - Buggy middle boxes (firewalls, proxies modifying ACK numbers)
# - Memory pressure on receiver causing buffer contents to be freed
# - Some NAT devices

# Fix: if reneging is from middlebox:
# Bypass the middlebox for this traffic path
# Or disable SACK (last resort - performance impact):
sysctl -w net.ipv4.tcp_sack=0
```

## SACK Scoreboard Analysis

```bash
# The SACK scoreboard tracks which segments were received out of order
# Problems occur when the scoreboard is wrong

# Enable detailed TCP tracing to see SACK scoreboard:
# (requires kernel debug build or eBPF)
bpftrace -e 'kprobe:tcp_sacktag_walk_frag { printf("SACK tag: %s\n", comm); }'

# Simpler: watch SACK events with tcpdump
tcpdump -i eth0 -n -v host 10.20.0.5 2>/dev/null | grep -i sack

# SACK block format in tcpdump output:
# sack 1 {1001:1500} = one SACK block, bytes 1001-1500 received
# sack 2 {2001:2500}{3001:3500} = two blocks (gap between 1500-2001 and 2500-3001)
```

## Performance Comparison With/Without SACK

```bash
# Test performance on a lossy link with and without SACK
# First, with SACK enabled (default):
sysctl -w net.ipv4.tcp_sack=1
iperf3 -c 10.20.0.5 -t 30 2>&1 | grep sender

# Introduce simulated loss:
tc qdisc add dev eth0 root netem loss 1%

# Test with SACK:
iperf3 -c 10.20.0.5 -t 30 2>&1 | grep sender

# Test without SACK:
sysctl -w net.ipv4.tcp_sack=0
iperf3 -c 10.20.0.5 -t 30 2>&1 | grep sender

# On 1% loss: SACK typically delivers 40-60% better throughput
# Clean up:
tc qdisc del dev eth0 root
sysctl -w net.ipv4.tcp_sack=1
```

## Conclusion

SACK is critical for performance on any link with packet loss. Verify both endpoints negotiate SACK in the handshake. Monitor `TcpExtTCPSACKReneging` — any non-zero value indicates a buggy middlebox stripping or modifying TCP options. DSACK counters tell you whether spurious retransmissions are occurring. Only disable SACK as a last resort; the performance cost on any path with >0.1% loss is significant.
