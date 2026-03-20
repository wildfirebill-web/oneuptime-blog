# How to Configure IPv6 Congestion Control Algorithms

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, TCP, Congestion Control, BBR, CUBIC, Linux, Performance

Description: Configure and compare TCP congestion control algorithms for IPv6 connections on Linux, including BBR, CUBIC, and HTCP, with per-connection and system-wide approaches.

## Introduction

TCP congestion control algorithms manage how quickly data is sent to avoid overwhelming the network. For IPv6 connections - particularly over high-bandwidth or satellite links - choosing the right algorithm can dramatically affect throughput and latency.

## Available Algorithms

```bash
# List all available congestion control modules

cat /proc/sys/net/ipv4/tcp_available_congestion_control

# Load all available modules
modprobe tcp_bbr
modprobe tcp_htcp
modprobe tcp_illinois
modprobe tcp_westwood

# Verify loaded modules
lsmod | grep tcp
```

## Step 1: Set System-Wide Congestion Control

```bash
# /etc/sysctl.d/99-congestion.conf

# Use BBR for most modern workloads (best for long-distance IPv6)
net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr

# This applies to all TCP connections, including IPv6
```

```bash
sudo sysctl -p /etc/sysctl.d/99-congestion.conf

# Verify the change
sysctl net.ipv4.tcp_congestion_control
```

## Step 2: Per-Connection Congestion Control in Applications

Applications can override the system-wide setting on a per-socket basis.

```python
import socket

# TCP_CONGESTION socket option (Linux constant = 13)
TCP_CONGESTION = 13

def create_ipv6_socket_with_cc(algorithm: str = "bbr"):
    """
    Create an IPv6 TCP socket with a specific congestion control algorithm.
    """
    sock = socket.socket(socket.AF_INET6, socket.SOCK_STREAM)

    # Set congestion control algorithm for this socket
    sock.setsockopt(
        socket.IPPROTO_TCP,
        TCP_CONGESTION,
        algorithm.encode()
    )

    return sock

# Example: use BBR for a latency-sensitive IPv6 connection
conn = create_ipv6_socket_with_cc("bbr")
conn.connect(("2001:db8::1", 443, 0, 0))
```

## Step 3: Algorithm Comparison

| Algorithm | Best For | Notes |
|---|---|---|
| BBR | Long-distance, lossy paths | Model-based; excellent throughput |
| CUBIC | LAN, high-bandwidth paths | Default on most Linux systems |
| HTCP | Mixed environments | Better than CUBIC on lossy links |
| Westwood | Wireless/mobile IPv6 | Handles burst losses well |
| Vegas | Low-RTT paths | Proactive; may under-utilize bandwidth |

## Step 4: Benchmark Different Algorithms

```bash
#!/bin/bash
# compare-cc.sh - benchmark congestion control algorithms

SERVER="2001:db8::1"
DURATION=30

for algo in bbr cubic htcp; do
  # Temporarily change algorithm
  sudo sysctl -w net.ipv4.tcp_congestion_control=$algo

  echo "=== $algo ==="
  iperf3 -6 -c "$SERVER" -t $DURATION -P 4 --format m 2>&1 | \
    grep "receiver"
done

# Restore BBR
sudo sysctl -w net.ipv4.tcp_congestion_control=bbr
```

## Step 5: Configure BBR with Optimal FQ Scheduler

BBR requires the Fair Queue (fq) packet scheduler to function optimally.

```bash
# Apply fq to all interfaces
for iface in $(ip -6 link show | awk -F': ' '/^[0-9]+/{print $2}' | grep -v lo); do
  # Set fq as the queueing discipline
  sudo tc qdisc replace dev "$iface" root fq
  echo "Set fq on $iface"
done

# Verify
tc qdisc show dev eth0

# Make persistent via a systemd service or /etc/rc.local
# Or add to /etc/network/interfaces post-up scripts
```

## Step 6: Monitor Congestion Events

```bash
# Watch for congestion window (cwnd) and RTT per connection
ss -6 -t -i | grep -A1 "ESTAB" | grep "cwnd\|rtt"

# Example output:
# cwnd:87 ssthresh:87 reordering:3 rtt:12.5/2.1 ato:40

# Track retransmission rate as a proxy for congestion
watch -n1 "netstat -s | grep -i retransmit"
```

## Conclusion

BBR is the recommended congestion control algorithm for IPv6 connections over any path with meaningful RTT or packet loss. Pair it with the `fq` qdisc for best results. Per-socket overrides let latency-sensitive applications use different algorithms. Monitor your IPv6 connection quality with OneUptime to detect when congestion is impacting user experience.
