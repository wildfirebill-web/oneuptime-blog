# How to Diagnose Packet Loss on an IPv4 Network

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Packet Loss, IPv4, Linux, Networking, Diagnostic, mtr

Description: Systematically diagnose IPv4 packet loss using ping, MTR, tcpdump, and interface statistics to identify whether loss occurs at the physical layer, a specific link, or the destination.

Packet loss causes slow downloads, dropped VoIP calls, and broken connections. But "packet loss" can occur for many different reasons at different points in the network path. This guide walks through a systematic diagnosis.

## Step 1: Confirm Packet Loss with Ping

```bash
# Run enough packets for statistically meaningful results

ping -c 100 8.8.8.8

# Look for: X packets transmitted, Y received, Z% packet loss
# Z > 0 = packet loss detected

# Test different destinations to localize the problem
ping -c 50 192.168.1.1     # Gateway (LAN)
ping -c 50 8.8.8.8         # Internet (DNS)
ping -c 50 1.1.1.1         # Different internet host

# Loss to gateway but not internet:
#   → Problem is between you and gateway
# Loss to all internet IPs:
#   → ISP link or gateway problem
# Loss to specific IP only:
#   → Remote end is having issues
```

## Step 2: Identify Which Hop Has Loss with MTR

```bash
# Continuous traceroute with per-hop loss statistics
sudo mtr --report --report-cycles=50 -n 8.8.8.8

# Key columns:
# Loss%: percentage of packets lost at this hop
# StDev: jitter (high = congestion)

# Interpretation:
# Loss at hop 3, zero loss at hop 4+ → hop 3 limits ICMP rate (not real loss)
# Loss at hop 3, same loss at hop 4+ → real loss starts at hop 3
```

## Step 3: Check Physical Layer Errors

```bash
# Check interface statistics for errors
ip -s link show eth0

# Output includes:
# RX: bytes  packets  errors  dropped  overrun  mcast
#     12345    100      0       0        0        0
# TX: bytes  packets  errors  dropped  carrier  collsns
#     23456    80       0       0        0        0

# RX errors > 0 → physical issues (bad cable, faulty NIC, duplex mismatch)
# TX errors > 0 → hardware issues
# dropped > 0   → buffer overflow (interface too slow for traffic)

# Also check with ethtool
sudo ethtool -S eth0 | grep -i error
```

## Step 4: Check for Duplex Mismatch

```bash
# Duplex mismatch is a common cause of high loss on LAN
sudo ethtool eth0 | grep -i duplex

# Expected: Full-duplex
# If "Half": check switch port configuration
# If mismatch (one side full, other half): high late collision errors

# Force full duplex (if auto-negotiation fails):
sudo ethtool -s eth0 duplex full speed 1000
```

## Step 5: Capture Loss Events with tcpdump

```bash
# Look for TCP retransmissions (evidence of loss)
sudo tcpdump -nn 'tcp[tcpflags] & tcp-syn != 0 or tcp[tcpflags] & tcp-rst != 0' -i eth0

# Count TCP retransmissions using tshark
sudo tshark -i eth0 -q -z io,stat,5,"tcp.analysis.retransmission"
# Shows retransmissions per 5-second interval

# High retransmissions + low interface errors = mid-path loss (ISP/WAN)
# High retransmissions + high interface errors = local physical problem
```

## Step 6: Check System Buffer Drops

```bash
# Check for UDP/socket drops (application buffer overflow)
netstat -s | grep "packet receive errors"

# Check socket receive buffer drops
ss -s | grep "buf"

# Increase receive buffer if needed
sudo sysctl -w net.core.rmem_max=16777216
sudo sysctl -w net.core.rmem_default=16777216
```

## Categorize and Fix

```text
Loss pattern                Likely cause             Fix
--------------------------  ----------------------   -------------------------
Loss at first hop           LAN problem              Cable, switch, NIC
Loss at gateway (100%)      Default route broken     Check/restart router
Loss beyond gateway         ISP issue                Contact ISP
Burst loss (not constant)   Congestion/bufferbloat   QoS, bandwidth upgrade
Loss at specific remote IP  Target host issues       Contact remote admin
Loss with high errors       Physical layer failure   Replace cable/NIC/switch
```

Systematic packet loss diagnosis avoids the common trap of blaming the wrong layer - physical errors and ISP congestion look identical from application logs but require completely different fixes.
