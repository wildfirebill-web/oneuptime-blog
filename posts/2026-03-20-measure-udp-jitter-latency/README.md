# How to Measure UDP Jitter and Latency on Your Network

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: UDP, Jitter, Latency, Performance, Networking, iperf3, Measurement

Description: Measure UDP one-way latency, round-trip time, and jitter using iperf3, custom scripts, and dedicated tools to assess network quality for real-time applications.

## Introduction

Latency and jitter are the critical metrics for real-time UDP applications. Latency is how long a packet takes to travel from sender to receiver. Jitter is the variation in latency between consecutive packets — high jitter causes audio dropouts and video stutters even when average latency is acceptable. VoIP requires jitter under 30ms, gaming requires under 10ms for smooth play.

## Measuring RTT with ping (ICMP baseline)

```bash
# ping gives RTT (round-trip) and jitter (mdev):
ping -c 100 -i 0.1 10.20.0.5 | tail -3
# rtt min/avg/max/mdev = 0.4/0.6/2.1/0.3 ms
# mdev = mean deviation = approximation of jitter

# Higher sample count for better jitter estimate:
ping -c 1000 -i 0.01 10.20.0.5 | tail -3
# mdev value = jitter in ms

# Note: ping uses ICMP, not UDP. Actual UDP jitter may differ.
# Use iperf3 UDP mode for UDP-specific measurement.
```

## UDP Jitter with iperf3

```bash
# Server (on remote host):
iperf3 -s

# Client: measure UDP jitter at low bitrate (simulates VoIP):
iperf3 -c 10.20.0.5 -u -b 1M -l 160 -t 60 --get-server-output
# -b 1M: 1 Mbps (simulates a single call leg)
# -l 160: 160-byte packets (G.711 VoIP payload)
# Output: Jitter column shows per-interval jitter in ms

# For gaming: small packets at gaming rate:
iperf3 -c 10.20.0.5 -u -b 100K -l 60 -t 30
# -l 60: small game state packets

# Interpret results:
# Jitter < 5ms: excellent (suitable for all real-time apps)
# Jitter 5-30ms: acceptable for VoIP (with jitter buffer)
# Jitter > 30ms: problematic for VoIP, unacceptable for gaming
```

## Custom UDP Latency Measurement

```python
#!/usr/bin/env python3
# udp_latency_measure.py - Measure one-way and RTT latency

import socket
import time
import statistics

SERVER = '10.20.0.5'
PORT = 5000
NUM_PACKETS = 100
INTERVAL = 0.05  # 50ms between packets

sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
sock.settimeout(1.0)

# Start echo server on remote:
# python3 -c "
# import socket
# s=socket.socket(socket.AF_INET,socket.SOCK_DGRAM)
# s.bind(('',5000))
# while True:
#     d,a=s.recvfrom(100)
#     s.sendto(d,a)
# "

rtts = []
for i in range(NUM_PACKETS):
    send_time = time.perf_counter()
    sock.sendto(str(send_time).encode(), (SERVER, PORT))
    try:
        data, _ = sock.recvfrom(100)
        rtt_ms = (time.perf_counter() - send_time) * 1000
        rtts.append(rtt_ms)
    except socket.timeout:
        pass
    time.sleep(INTERVAL)

sock.close()

if rtts:
    print(f"Packets: {len(rtts)}/{NUM_PACKETS}")
    print(f"Min RTT:  {min(rtts):.2f} ms")
    print(f"Avg RTT:  {statistics.mean(rtts):.2f} ms")
    print(f"Max RTT:  {max(rtts):.2f} ms")
    jitter = statistics.stdev(rtts)
    print(f"Jitter:   {jitter:.2f} ms (std dev of RTT)")

    # Percentiles:
    sorted_rtts = sorted(rtts)
    p99 = sorted_rtts[int(0.99 * len(sorted_rtts))]
    print(f"P99 RTT:  {p99:.2f} ms")
```

## Continuous Jitter Monitoring

```bash
#!/bin/bash
# Monitor jitter over time, log results

SERVER="10.20.0.5"
LOG_FILE="/var/log/udp-jitter.log"

while true; do
    RESULT=$(iperf3 -c $SERVER -u -b 1M -l 160 -t 10 -J 2>/dev/null)
    JITTER=$(echo "$RESULT" | python3 -c "
import json,sys
try:
    d=json.load(sys.stdin)
    print(f\"{d['end']['sum']['jitter_ms']:.3f}\")
except: print('error')")
    LOSS=$(echo "$RESULT" | python3 -c "
import json,sys
try:
    d=json.load(sys.stdin)
    print(f\"{d['end']['sum']['lost_percent']:.2f}\")
except: print('error')")
    echo "$(date +%Y-%m-%dT%H:%M:%S) jitter=${JITTER}ms loss=${LOSS}%" | tee -a $LOG_FILE
    sleep 60
done
```

## Network Quality Thresholds

```
Metric       | Excellent | Acceptable | Poor
-------------|-----------|------------|------
RTT          | < 20ms    | 20-100ms   | > 100ms
Jitter       | < 5ms     | 5-30ms     | > 30ms
Packet Loss  | < 0.1%    | 0.1-1%     | > 1%

Application requirements:
VoIP (G.711):     RTT < 150ms, Jitter < 30ms, Loss < 1%
HD Video call:    RTT < 100ms, Jitter < 15ms, Loss < 0.5%
Online gaming:    RTT < 30ms,  Jitter < 10ms, Loss < 0.1%
Live streaming:   RTT < 3s,    Jitter < 50ms, Loss < 0.5%
```

## Conclusion

iperf3 UDP mode provides the most accurate jitter measurement — run with small packets at the target bitrate to simulate your specific application. Monitor `mdev` from ping for quick jitter checks. For production monitoring, run the jitter logging script continuously and alert when jitter exceeds application-specific thresholds. P99 latency is more actionable than average for user-facing applications — a P99 > 50ms is noticeable even when average is low.
