# How to Tune TCP Buffer Sizes on Linux for High Throughput

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: TCP, Linux, Performance, Sysctl, Network Tuning, Throughput

Description: Learn how to tune Linux TCP buffer sizes using sysctl parameters to maximize throughput on high-bandwidth or high-latency network connections.

## Why TCP Buffer Sizes Matter

TCP needs buffer space to hold data in flight. The maximum throughput achievable on a TCP connection is:

```text
Max Throughput = TCP Window Size / Round-Trip Time
```

If the buffer is too small, TCP throttles the sender even when bandwidth is available. On a 1 Gbps link with 100ms RTT, you need at least 12.5 MB of buffer to fill the pipe.

## Key TCP Buffer Parameters

| Parameter | Description | Default (typical) |
|---|---|---|
| `net.core.rmem_max` | Max socket receive buffer | 212992 bytes (~208 KB) |
| `net.core.wmem_max` | Max socket send buffer | 212992 bytes |
| `net.ipv4.tcp_rmem` | TCP receive buffer: min/default/max | 4096 87380 6291456 |
| `net.ipv4.tcp_wmem` | TCP send buffer: min/default/max | 4096 16384 4194304 |

## Step 1: Check Current Buffer Settings

```bash
# View current settings

sysctl net.core.rmem_max
sysctl net.core.wmem_max
sysctl net.ipv4.tcp_rmem
sysctl net.ipv4.tcp_wmem

# Check if auto-tuning is enabled (should be 1)
sysctl net.ipv4.tcp_moderate_rcvbuf
```

## Step 2: Calculate Required Buffer Size

For high-throughput tuning, calculate the Bandwidth-Delay Product (BDP):

```bash
# BDP = Bandwidth × RTT
# For 10 Gbps link with 50ms RTT:
# BDP = 10,000,000,000 bits/sec × 0.050 sec = 500,000,000 bits = 62.5 MB

# Rule: Set max buffer to at least 2× BDP for headroom
# Buffer needed = 125 MB per connection
```

## Step 3: Apply TCP Buffer Tuning

```bash
# Create or edit /etc/sysctl.d/99-tcp-tuning.conf

cat > /etc/sysctl.d/99-tcp-tuning.conf << 'EOF'
# Maximum socket buffer sizes (must be >= tcp_rmem/wmem max)
net.core.rmem_max = 134217728     # 128 MB
net.core.wmem_max = 134217728     # 128 MB

# Default socket buffer sizes (for non-TCP sockets)
net.core.rmem_default = 25165824  # 24 MB
net.core.wmem_default = 25165824  # 24 MB

# TCP buffer sizes: min / default / max
# Auto-tuning uses these limits
net.ipv4.tcp_rmem = 4096 1048576 134217728
#                   ^^^^  ^^^^^^^  ^^^^^^^^^
#                   min   default  max (128MB)
net.ipv4.tcp_wmem = 4096 1048576 134217728

# Enable TCP memory auto-tuning (default on most systems)
net.ipv4.tcp_moderate_rcvbuf = 1
EOF

# Apply settings immediately
sudo sysctl -p /etc/sysctl.d/99-tcp-tuning.conf
```

## Step 4: Enable TCP Auto-Tuning

Linux automatically adjusts buffers within the configured range when auto-tuning is enabled:

```bash
# Ensure auto-tuning is on (should be by default)
sysctl net.ipv4.tcp_moderate_rcvbuf

# If disabled, enable it
sudo sysctl -w net.ipv4.tcp_moderate_rcvbuf=1
```

## Step 5: Set System-Wide Memory Limits for TCP

The `net.ipv4.tcp_mem` parameter controls how much total memory TCP can use across all connections:

```bash
# tcp_mem: min/pressure/max in PAGES (typically 4096 bytes each)
# For 128GB system:
# min = 1GB / 4096 = 262144 pages
# pressure = 2GB / 4096 = 524288 pages
# max = 4GB / 4096 = 1048576 pages

echo 'net.ipv4.tcp_mem = 262144 524288 1048576' >> /etc/sysctl.d/99-tcp-tuning.conf
sudo sysctl -p /etc/sysctl.d/99-tcp-tuning.conf
```

## Step 6: Verify with iperf3

Test throughput before and after tuning:

```bash
# Install iperf3
sudo apt-get install -y iperf3

# On server side
iperf3 -s -p 5201

# On client side - test before tuning
iperf3 -c server-ip -t 30 -P 4   # 4 parallel streams

# After tuning, re-run and compare
# Expected: significant throughput increase on high-BDP links
```

## Step 7: Per-Application Buffer Tuning

For applications that need large buffers, set socket options in the application code:

```python
import socket

sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
# Set socket receive buffer to 64 MB
sock.setsockopt(socket.SOL_SOCKET, socket.SO_RCVBUF, 64 * 1024 * 1024)
sock.setsockopt(socket.SOL_SOCKET, socket.SO_SNDBUF, 64 * 1024 * 1024)
```

Note: The socket buffer cannot exceed `net.core.rmem_max`/`wmem_max`.

## Conclusion

TCP buffer tuning on Linux involves setting `net.core.rmem_max`, `wmem_max`, and `net.ipv4.tcp_rmem/wmem` to values large enough to fill the network pipe. Calculate the required buffer as 2× the Bandwidth-Delay Product, persist settings in `/etc/sysctl.d/`, and verify improvement with iperf3. Keep TCP auto-tuning enabled so buffers grow only as needed.
