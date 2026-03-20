# How to Configure TCP Buffer Sizes for Maximum Throughput

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: TCP, Linux, Performance, Buffer Sizes, Throughput, Kernel Tuning

Description: Configure Linux TCP send and receive buffer sizes to maximize throughput for bulk data transfers, using bandwidth-delay product calculations as the baseline.

## Introduction

TCP buffer sizes determine how much data can be in flight between sender and receiver at any time. Setting buffers appropriately for your network's bandwidth-delay product is the most impactful single change for improving TCP throughput. Too small buffers leave bandwidth unused; excessively large buffers waste memory and can cause bufferbloat.

## Complete TCP Buffer Configuration

```bash
# Calculate your bandwidth-delay product first
# BDP = bandwidth (bytes/sec) × RTT (seconds)

# Example: 10 Gbps link, 5ms RTT
# BDP = (10,000,000,000 / 8) × 0.005 = 6,250,000 bytes ≈ 6 MB

# Apply comprehensive buffer tuning
# (replace values with your calculated BDP × 2 for headroom)

# Send and receive buffers
sysctl -w net.ipv4.tcp_rmem="4096 131072 12582912"   # max = 12MB (2× BDP)
sysctl -w net.ipv4.tcp_wmem="4096 131072 12582912"

# System-wide socket buffer limits (must be >= tcp_rmem/wmem max)
sysctl -w net.core.rmem_max=12582912
sysctl -w net.core.wmem_max=12582912
sysctl -w net.core.rmem_default=131072
sysctl -w net.core.wmem_default=131072

# Enable auto-tuning
sysctl -w net.ipv4.tcp_window_scaling=1
sysctl -w net.ipv4.tcp_moderate_rcvbuf=1
sysctl -w net.core.netdev_max_backlog=5000

# Persist all settings
cat > /etc/sysctl.d/tcp-performance.conf << EOF
# TCP buffer tuning for high-throughput
net.ipv4.tcp_rmem=4096 131072 12582912
net.ipv4.tcp_wmem=4096 131072 12582912
net.core.rmem_max=12582912
net.core.wmem_max=12582912
net.core.rmem_default=131072
net.core.wmem_default=131072
net.ipv4.tcp_window_scaling=1
net.ipv4.tcp_moderate_rcvbuf=1
net.core.netdev_max_backlog=5000
EOF
sysctl -p /etc/sysctl.d/tcp-performance.conf
```

## Testing Before and After

```bash
# Baseline measurement (run iperf3 server on remote host first)
# Remote: iperf3 -s
iperf3 -c 10.20.0.5 -t 30 -P 4   # 4 parallel streams

# After applying tuning:
iperf3 -c 10.20.0.5 -t 30 -P 4

# Compare: sender/receiver bitrates, retransmits, sender window size
```

## Per-Application Tuning

For specific high-throughput applications, override system defaults:

```python
import socket

def create_optimized_socket(buffer_mb=8):
    """Create a TCP socket with large buffers for bulk transfers."""
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

    buffer_bytes = buffer_mb * 1024 * 1024

    # Set send buffer
    s.setsockopt(socket.SOL_SOCKET, socket.SO_SNDBUF, buffer_bytes)

    # Set receive buffer
    s.setsockopt(socket.SOL_SOCKET, socket.SO_RCVBUF, buffer_bytes)

    # Verify (OS caps at rmem_max / 2 for some kernels)
    actual_send = s.getsockopt(socket.SOL_SOCKET, socket.SO_SNDBUF)
    actual_recv = s.getsockopt(socket.SOL_SOCKET, socket.SO_RCVBUF)
    print(f"Send buffer: {actual_send/1024/1024:.1f} MB")
    print(f"Receive buffer: {actual_recv/1024/1024:.1f} MB")

    return s
```

## Avoiding Bufferbloat

```bash
# Too-large buffers cause bufferbloat: high latency under load
# Don't set buffers much larger than 2× BDP for your bottleneck link

# Check if bufferbloat is present: measure latency during bulk transfer
# Terminal 1: start transfer
iperf3 -c 10.20.0.5 -t 60

# Terminal 2: measure latency during transfer (should stay low)
ping -c 60 -i 1 10.20.0.5

# If ping RTT triples during iperf: bufferbloat is present
# Fix: use BBR congestion control or reduce buffer max
sysctl -w net.ipv4.tcp_congestion_control=bbr
```

## Conclusion

TCP buffer sizing is a balance between throughput (larger is better) and latency (smaller avoids bufferbloat). Start with BDP × 2 as your target max buffer. For most modern servers with typical network latencies (1-50ms), values between 4MB and 16MB strike the right balance. Always measure before and after, and check that latency under load doesn't increase significantly after tuning.
