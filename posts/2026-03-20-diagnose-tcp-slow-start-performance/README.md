# How to Diagnose TCP Slow Start Performance Issues

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: TCP, Slow Start, Performance, Networking, Congestion Control, Linux

Description: Understand how TCP slow start limits throughput for short-lived connections and learn techniques to mitigate its impact on small file transfers.

## Introduction

TCP slow start is the initial phase of congestion control where the sender begins with a small congestion window and doubles it each RTT until it reaches a threshold or detects congestion. For long-lived bulk transfers, slow start is a brief startup cost. But for short connections — web requests, API calls, database queries — slow start may dominate the entire transfer time, severely limiting effective throughput.

## Understanding Slow Start Impact

```
Example: HTTP request for a 100KB file, RTT = 50ms

Without slow start issues:
Time = 100KB / 1Gbps = 0.8ms (negligible)

With slow start (initial window = 10 MSS = 14.6KB):
RTT 1: 14.6 KB sent
RTT 2: 29.2 KB sent (window doubled)
RTT 3: 56 KB sent (reaches threshold)
Total time: 3 × 50ms = 150ms for 100KB!
Effective throughput: 100KB / 0.150s = 5.3 Mbps (out of 1 Gbps available!)
```

## Measuring Slow Start's Impact

```bash
# Time a small file download to see slow start dominance
time curl -o /dev/null http://10.20.0.5/100kb.bin -w "time_total: %{time_total}s\n"

# Compare with multiple sequential requests (later ones benefit from cached windows)
time (for i in {1..5}; do curl -o /dev/null -s http://10.20.0.5/100kb.bin; done)
# Later requests are faster if connection is reused (HTTP keep-alive bypasses slow start)
```

## Initial Congestion Window Size

```bash
# Check current initial congestion window
ss -tin state established | grep "snd_cwnd" | head -5
# snd_cwnd at very start of connection = initial window

# View kernel's initial congestion window per route
ip route show | grep initcwnd
# default via 192.168.1.1 dev eth0 proto static metric 100
# If no initcwnd shown: default is 10 MSS (modern Linux)

# Increase initial window for specific routes
ip route change default via 192.168.1.1 initcwnd 32
# initcwnd 32 = 32 MSS = 46.7KB initial window

# Check available bandwidth estimate for initial window
ip route show
```

## Increasing Initial Congestion Window

```bash
# For local routes where you know bandwidth is available
ip route change 10.0.0.0/8 via 10.0.0.1 initcwnd 32

# For the default route (use carefully — only if you have excess bandwidth)
ip route change default via 192.168.1.1 initcwnd 32

# To see if kernel supports initcwnd
ip route change default via 192.168.1.1 initcwnd 32 2>&1
# "RTNETLINK answers: Invalid argument" = not supported by this kernel version
```

## TCP Slow Start After Idle (ssthresh Retention)

```bash
# After a connection is idle, Linux may reset CWND to initcwnd
# This is "slow start after idle"

# Check current setting
sysctl net.ipv4.tcp_slow_start_after_idle
# Default: 1 (enabled — resets CWND after idle period)

# Disable for persistent connections (HTTP keep-alive, WebSockets)
# This preserves the CWND from the previous transfer
sysctl -w net.ipv4.tcp_slow_start_after_idle=0

# Persist
echo "net.ipv4.tcp_slow_start_after_idle=0" >> /etc/sysctl.conf
```

## Application-Level Mitigations

```python
# Use connection pooling to avoid repeated slow starts
import requests

# Session with connection reuse — avoids slow start on subsequent requests
session = requests.Session()
session.mount('http://', requests.adapters.HTTPAdapter(
    pool_connections=5,
    pool_maxsize=10,
    max_retries=3
))

# All requests through this session reuse existing TCP connections
for url in urls:
    resp = session.get(url)   # No slow start after first request
```

## Conclusion

TCP slow start is most impactful for short-lived connections on high-bandwidth links. The fixes are: increase `initcwnd` for routes where bandwidth is plentiful (carefully), disable `tcp_slow_start_after_idle` for persistent connections, and use connection pooling in applications to avoid repeated slow-start ramp-ups. For very latency-sensitive services, TCP Fast Open also helps by combining the handshake with the first request.
