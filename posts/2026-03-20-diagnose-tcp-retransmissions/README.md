# How to Diagnose TCP Retransmissions and Window Zero Events

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: TCP, Retransmission, Window Zero, Wireshark, tcpdump, Performance

Description: Learn how to identify and diagnose TCP retransmissions and window zero conditions using command-line tools and Wireshark, and determine whether the root cause is packet loss or application slowness.

## What Cause TCP Retransmissions and Window Zero?

**TCP retransmissions** occur when a sent segment is not acknowledged within the retransmission timeout (RTO). Causes:
- Packet loss (network congestion, flapping links)
- High latency causing timeouts before ACK arrives
- Firewall dropping packets silently

**Window Zero** occurs when the receiver's buffer fills up and it advertises a zero receive window. The sender must stop transmitting until the receiver sends a window update. Causes:
- Application not reading data fast enough
- Insufficient receive buffer configuration
- Slow application processing

## Step 1: Check Global Retransmission Counters

```bash
# Quick retransmission count

netstat -s | grep -i "retransmit\|window"

# Output:
#     12456 segments retransmitted
#     0 retransmits while in timeout state
#     0 out of window
#     32 connections reset due to unexpected data

# Or with ss
ss -s

# Using nstat for precise counters
nstat -az | grep -E "TcpRetrans|TcpInErrs|TcpExtTCPLostRetransmit"

# Key counters:
# TcpRetransSegs - total retransmissions
# TcpExtTCPTimeouts - retransmit due to timeout (bad: indicates loss or RTT issue)
# TcpExtTCPFastRetrans - fast retransmit (normal: triggered by duplicate ACKs)
# TcpExtTCPSackFailures - SACK-based retransmit failures
```

## Step 2: Monitor Live Retransmissions with ss

```bash
# Watch active TCP connections for retransmissions
watch -n 1 "ss -tin | grep -E 'Retrans|rto|rtt'"

# Show connections with non-zero retransmission count
ss -tin | awk '/retrans=[^0]/ {print}'

# Key fields in ss -tin output:
# rto:200      - current retransmission timeout in ms
# rtt:1.5/0.5  - mean RTT / smoothed standard deviation
# Retrans:0/3  - fast retransmits / timeout retransmits
```

## Step 3: Capture and Analyze with tcpdump

```bash
# Capture all traffic on port 443 for 60 seconds
sudo tcpdump -i eth0 -w /tmp/retrans-capture.pcap port 443

# Analyze for retransmissions (duplicate sequence numbers)
tshark -r /tmp/retrans-capture.pcap -Y "tcp.analysis.retransmission or tcp.analysis.fast_retransmission" \
  -T fields -e frame.time -e ip.src -e ip.dst -e tcp.seq -e tcp.analysis.retransmission

# Check for window zero events
tshark -r /tmp/retrans-capture.pcap -Y "tcp.window_size == 0" \
  -T fields -e frame.time -e ip.src -e ip.dst -e tcp.window_size
```

## Step 4: Wireshark TCP Analysis

In Wireshark:

1. Open the PCAP file
2. Go to **Statistics → TCP Stream Graphs → Time-Sequence Graph (tcptrace)**
3. Look for:
   - **Vertical drops in the sequence graph** - retransmissions
   - **Flat periods (no sequence advance)** - window zero stalls
4. Use display filter: `tcp.analysis.flags` to highlight all TCP anomalies

```text
# Useful Wireshark display filters:
tcp.analysis.retransmission          - retransmitted segments
tcp.analysis.fast_retransmission     - 3 dup ACK triggered retransmit
tcp.analysis.duplicate_ack           - duplicate ACKs (indicate loss)
tcp.window_size == 0                 - window zero (receiver full)
tcp.analysis.zero_window             - window zero events
tcp.analysis.zero_window_probe       - probes sent after window zero
```

## Step 5: Distinguish Loss vs Application Slow

```bash
# Check if retransmissions correlate with network loss
ping -f -c 1000 <destination-ip>   # Fast ping to check loss %

# If packet loss > 0.1%, it's likely network congestion
# If packet loss = 0 but window zero is common, it's application slowness

# Check application processing lag
# For web servers:
cat /var/log/nginx/access.log | awk '{print $NF}' | sort -n | tail -20
# High request times = application is slow, causing window zero
```

## Step 6: Reduce Retransmissions

```bash
# If caused by packet loss - upgrade/fix the network
# If caused by high latency - adjust RTO parameters

# Reduce minimum RTO (cautiously)
sudo sysctl -w net.ipv4.tcp_rto_min_us=10000   # 10ms minimum (default ~200ms)

# Enable ECN to signal congestion without dropping
sudo sysctl -w net.ipv4.tcp_ecn=1

# If caused by window zero - increase receive buffers
sudo sysctl -w net.ipv4.tcp_rmem="4096 1048576 134217728"
```

## Conclusion

TCP retransmissions are caused by packet loss (network problems) while window zero events are caused by slow application processing. Distinguish between them using `nstat TcpExtTCPTimeouts` for loss-based retransmits, `tcp.window_size == 0` in Wireshark for application stalls, and `ss -tin` for per-connection retransmit counts. Address loss retransmissions by fixing the network path; address window zero by increasing buffers and optimizing the receiving application.
