# How to Configure UDP Timeout Values on Linux

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: UDP, Timeout, Linux, Socket, Networking, Configuration

Description: Configure UDP socket timeout values for receive operations, connection tracking, and application-level timeouts to prevent hanging UDP operations.

## Introduction

UDP sockets can block indefinitely on `recvfrom()` if no packets arrive. Unlike TCP, there is no connection-level timeout to fall back on. Configuring appropriate timeouts at the socket level prevents applications from hanging, while kernel-level conntrack timeouts control how long stateful firewalls track UDP flows. This guide covers both.

## Socket-Level Receive Timeout

```bash
# Application-level: set socket receive timeout
# This is the most important UDP timeout for applications

# Python example:
python3 << 'EOF'
import socket

sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
sock.bind(('0.0.0.0', 5000))

# Set 2-second timeout on recvfrom():
sock.settimeout(2.0)

try:
    data, addr = sock.recvfrom(1024)
    print(f"Received: {data}")
except socket.timeout:
    print("Timed out waiting for data")

sock.close()
EOF
```

## Setting Timeout with setsockopt

```bash
# Lower-level: use SO_RCVTIMEO socket option
# Equivalent to settimeout() but at C/syscall level

python3 << 'EOF'
import socket
import struct

sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
sock.bind(('0.0.0.0', 5001))

# Set SO_RCVTIMEO to 1 second, 500000 microseconds = 1.5 seconds
timeval = struct.pack('ll', 1, 500000)
sock.setsockopt(socket.SOL_SOCKET, socket.SO_RCVTIMEO, timeval)

try:
    data, addr = sock.recvfrom(65535)
    print(f"Received from {addr}: {data}")
except BlockingIOError:  # EAGAIN returned when timeout expires
    print("Receive timeout")

sock.close()
EOF
```

## Application-Level Retry with Timeout

```bash
# UDP client with timeout and retry logic:
python3 << 'EOF'
import socket
import time

def udp_request_with_retry(server, port, data, retries=3, timeout=1.0):
    sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    sock.settimeout(timeout)

    for attempt in range(retries):
        try:
            sock.sendto(data.encode(), (server, port))
            response, _ = sock.recvfrom(65535)
            sock.close()
            return response
        except socket.timeout:
            print(f"Attempt {attempt+1}/{retries} timed out")
            time.sleep(0.1 * (2 ** attempt))  # Exponential backoff

    sock.close()
    return None

result = udp_request_with_retry('10.20.0.5', 5000, 'query', retries=3, timeout=1.0)
if result:
    print(f"Response: {result}")
else:
    print("All retries failed")
EOF
```

## Kernel Connection Tracking (conntrack) Timeout

```bash
# For stateful firewalls using nf_conntrack:
# UDP connections are tracked even though UDP has no real connection state

# Check current conntrack UDP timeouts:
sysctl net.netfilter.nf_conntrack_udp_timeout
# Default: 30 seconds (for non-assured connections)
sysctl net.netfilter.nf_conntrack_udp_timeout_stream
# Default: 120 seconds (for flows with traffic in both directions)

# For long-lived UDP streams (RTP, DNS resolvers):
# Increase to prevent conntrack from expiring active flows:
sysctl -w net.netfilter.nf_conntrack_udp_timeout_stream=180

# For short-lived query/response UDP (DNS queries, NTP):
# Keep timeout short to reclaim conntrack entries quickly:
sysctl -w net.netfilter.nf_conntrack_udp_timeout=20

# Check total conntrack entries:
cat /proc/sys/net/netfilter/nf_conntrack_count
# If close to nf_conntrack_max: conntrack table is full
sysctl net.netfilter.nf_conntrack_max
```

## Non-Blocking UDP with Poll/Select

```bash
# Use select() instead of blocking with timeout:
python3 << 'EOF'
import socket
import select

sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
sock.bind(('0.0.0.0', 5002))

TIMEOUT = 2.0  # seconds

while True:
    ready, _, _ = select.select([sock], [], [], TIMEOUT)
    if ready:
        data, addr = sock.recvfrom(65535)
        print(f"Received: {data}")
    else:
        print("Timeout - no data received")
        break  # or continue polling

sock.close()
EOF
```

## Conclusion

UDP socket timeouts are entirely application-controlled. Use `sock.settimeout(seconds)` for the simplest implementation, or `SO_RCVTIMEO` via `setsockopt` for lower-level control. Implement retry logic with exponential backoff for request/response UDP protocols like DNS or NTP. For kernel-level firewall timeout, tune `nf_conntrack_udp_timeout_stream` to match your application's traffic pattern — too short causes flows to be dropped from the conntrack table while still active.
