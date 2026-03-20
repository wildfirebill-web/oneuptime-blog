# How to Switch TCP Congestion Control Algorithms on Linux

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: TCP, Linux, Congestion Control, BBR, CUBIC, Kernel

Description: Change the TCP congestion control algorithm on Linux at the system level or per-connection, and verify the change is active.

## Introduction

Linux allows you to change the TCP congestion control algorithm system-wide or per-socket. The system default applies to all new connections unless overridden at the application level. Switching algorithms — particularly from CUBIC to BBR — can dramatically improve throughput on high-latency or lossy network paths.

## Viewing Available Algorithms

```bash
# List all available algorithms
sysctl net.ipv4.tcp_available_congestion_control
# cubic reno bbr ...

# If BBR is not listed, load the module
modprobe tcp_bbr
sysctl net.ipv4.tcp_available_congestion_control  # Check again

# View currently active algorithm
sysctl net.ipv4.tcp_congestion_control
```

## Loading Additional Congestion Control Modules

```bash
# Load built-in algorithms (kernel modules)
modprobe tcp_bbr       # BBR
modprobe tcp_htcp      # H-TCP (good for high-speed links)
modprobe tcp_westwood  # Better for wireless
modprobe tcp_hybla     # Better for satellite links

# Auto-load on boot
echo "tcp_bbr" >> /etc/modules

# Verify module loaded
lsmod | grep tcp
```

## Changing System-Wide Algorithm

```bash
# Switch to BBR
sysctl -w net.ipv4.tcp_congestion_control=bbr

# Verify
sysctl net.ipv4.tcp_congestion_control
# net.ipv4.tcp_congestion_control = bbr

# Make permanent (survives reboot)
echo "net.ipv4.tcp_congestion_control=bbr" >> /etc/sysctl.conf
sysctl -p

# Or use sysctl.d for cleaner organization
echo "net.ipv4.tcp_congestion_control=bbr" > /etc/sysctl.d/10-tcp-bbr.conf
sysctl -p /etc/sysctl.d/10-tcp-bbr.conf
```

## Per-Socket Algorithm Override

Applications can override the system default for their specific sockets:

```python
# Python: set congestion control per socket (Linux-specific)
import socket

TCP_CONGESTION = 13  # Linux socket option

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

# Set BBR for this specific socket
s.setsockopt(socket.IPPROTO_TCP, TCP_CONGESTION, b'bbr\x00')

# Verify
algo = s.getsockopt(socket.IPPROTO_TCP, TCP_CONGESTION, 16)
print(f"Algorithm: {algo.rstrip(b'\\x00').decode()}")
```

```bash
# ss shows the congestion control algorithm per connection
ss -tin state established | grep -oP 'cubic|bbr|reno'
```

## Switching Algorithm and Testing

```bash
#!/bin/bash
# Compare algorithms on your network
TARGET="10.20.0.5"

echo "=== Testing TCP Congestion Control Algorithms ==="
for algo in cubic bbr reno; do
    # Check if algorithm is available
    if echo "$algo" | grep -qF "$(sysctl -n net.ipv4.tcp_available_congestion_control)"; then
        sysctl -w net.ipv4.tcp_congestion_control=$algo >/dev/null 2>&1
        RESULT=$(iperf3 -c $TARGET -t 10 -J 2>/dev/null | \
                 python3 -c "import sys,json; d=json.load(sys.stdin); \
                   print(f'{d[\"end\"][\"sum_sent\"][\"bits_per_second\"]/1e6:.1f} Mbps')")
        echo "$algo: $RESULT"
    fi
done
```

## Verifying the Algorithm in Active Connections

```bash
# Check algorithm for all established connections
ss -tino state established | grep "cc:"

# Or for a specific connection
ss -tino "( dst 10.20.0.5 )"
# Look for: cc:bbr (or cc:cubic)

# Per-connection statistics with current algorithm
ss -tin state established | head -30
```

## Conclusion

Changing congestion control on Linux is a one-line sysctl command. Load the desired algorithm's module first (`modprobe tcp_bbr`), set it as the default, and persist in `/etc/sysctl.conf`. Applications can also override per-socket using `TCP_CONGESTION`. Run a brief iperf3 comparison test before committing to a change in production — the right algorithm depends on your specific network characteristics.
