# How to Configure UDP Buffer Sizes on Linux

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: UDP, Buffer, Linux, Performance, sysctl, Kernel

Description: Configure UDP socket receive and send buffer sizes at the kernel and application level to prevent packet drops in high-throughput UDP applications.

## Introduction

UDP packet loss due to buffer overflow is one of the most common UDP performance problems. When packets arrive faster than the application reads them, the kernel drops them silently. The fix is a two-part process: increase the kernel's maximum allowed buffer size with `sysctl`, and then set the actual socket buffer size in the application with `SO_RCVBUF`.

## Understanding UDP Buffers

```
UDP receive path:
  NIC → kernel ring buffer → socket receive buffer → application read

UDP send path:
  application write → socket send buffer → kernel → NIC

Drops occur when:
  - Socket receive buffer full: incoming packets dropped by kernel
    Counter: UdpRcvbufErrors (nstat)
  - Socket send buffer full: sendto() blocks or returns ENOBUFS
    Counter: UdpSndbufErrors (nstat)
```

## Check Current Buffer Settings

```bash
# Kernel maximums (ceiling for what applications can request)
sysctl net.core.rmem_max    # Max receive buffer size
sysctl net.core.wmem_max    # Max send buffer size
sysctl net.core.rmem_default  # Default receive buffer size

# Check actual socket buffer usage:
ss -umn  # UDP sockets with memory info
# "r" column: receive bytes queued
# Shows socket-level buffer usage

# Check if drops are occurring:
netstat -su | grep -E "error|overflow|buffer"
# Or:
nstat | grep -E "UdpRcvbuf|UdpSndbuf|UdpInErrors"
```

## Increase Kernel Buffer Limits

```bash
# Temporary change (reverts on reboot):
sysctl -w net.core.rmem_max=26214400     # 25 MB max receive
sysctl -w net.core.wmem_max=26214400     # 25 MB max send
sysctl -w net.core.rmem_default=8388608  # 8 MB default receive

# Permanent change (persists across reboots):
cat >> /etc/sysctl.conf << 'EOF'
net.core.rmem_max = 26214400
net.core.wmem_max = 26214400
net.core.rmem_default = 8388608
net.core.wmem_default = 8388608
EOF
sysctl -p

# These are the kernel maximum values; applications can request
# up to this limit via SO_RCVBUF/SO_SNDBUF socket options
```

## Set Buffer Size in Application

```bash
# After increasing kernel limits, the application must also request
# the larger buffer size. The default socket buffer is rmem_default.

# Python example:
cat << 'EOF' > /tmp/udp_server_large_buf.py
import socket

sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)

# Request 25 MB receive buffer
sock.setsockopt(socket.SOL_SOCKET, socket.SO_RCVBUF, 26214400)

# Verify what kernel actually allocated (may be less than requested)
actual = sock.getsockopt(socket.SOL_SOCKET, socket.SO_RCVBUF)
print(f"Requested: 25MB, Got: {actual // 1024 // 1024}MB")
# Note: kernel doubles the value internally, so 25MB request → 50MB shown

sock.bind(('0.0.0.0', 5000))
print("Listening on :5000")
EOF
python3 /tmp/udp_server_large_buf.py
```

## Force Larger Buffers with SO_RCVBUFFORCE

```bash
# If rmem_max is too low and you can't change it:
# Root can use SO_RCVBUFFORCE to override the limit

# Python (requires root):
import socket
SO_RCVBUFFORCE = 33  # Constant value

sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
sock.setsockopt(socket.SOL_SOCKET, SO_RCVBUFFORCE, 67108864)  # 64 MB
```

## Calculate Required Buffer Size

```bash
# Formula: buffer >= arrival_rate * processing_time
# Example: 100K packets/sec, 5ms processing burst:
# buffer = 100000 * 0.005 = 500 packets during burst
# At 1500 bytes each: 500 * 1500 = 750 KB minimum

# For high-throughput video streaming at 100 Mbps, 1 second buffer:
# buffer = 100 Mbps * 1s = 12.5 MB

python3 -c "
rate_mbps = 100
burst_seconds = 0.1  # 100ms burst
buffer_bytes = rate_mbps * 1e6 / 8 * burst_seconds
print(f'Recommended buffer: {buffer_bytes/1024/1024:.1f} MB')
"
```

## Conclusion

UDP buffer configuration is a two-step process: set the kernel maximum with `sysctl net.core.rmem_max`, then request the actual size in the application with `SO_RCVBUF`. Monitor `UdpRcvbufErrors` in `nstat` to confirm drops have stopped. Size the buffer to handle your expected burst duration: multiply the arrival rate by the maximum burst duration to get the minimum buffer size needed.
