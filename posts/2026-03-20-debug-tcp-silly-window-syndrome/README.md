# How to Debug TCP Silly Window Syndrome

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: TCP, Networking, Silly Window Syndrome, Performance, Nagle, Clark's Algorithm

Description: Understand TCP Silly Window Syndrome (SWS), how it causes inefficient small segment transmission, and how Nagle's algorithm and Clark's algorithm prevent it.

## Introduction

Silly Window Syndrome (SWS) occurs when TCP sends or advertises very small amounts of data, leading to network inefficiency where overhead (TCP/IP headers: 40 bytes) greatly exceeds the useful payload (perhaps 1-2 bytes). This is particularly harmful for interactive applications like telnet or terminals where a single keystroke generates a packet.

## The Two Forms of SWS

**Sender SWS**: Sender sends small segments when it should wait for more data to accumulate.

**Receiver SWS**: Receiver advertises very small window updates, causing sender to send tiny segments to fill them.

## Detecting SWS in a Packet Capture

```bash
# Capture and look for very small TCP segments

tcpdump -i eth0 -n -v 'tcp' 2>/dev/null | \
  awk '/length/ {
    match($0, /length ([0-9]+)/, a)
    if (a[1]+0 > 0 && a[1]+0 < 20) print "SMALL SEGMENT ("a[1]" bytes):", $0
  }'

# In Wireshark: filter for small TCP data segments
# tcp.len > 0 && tcp.len < 20
# Large numbers of these indicate SWS
```

## Nagle's Algorithm (Sender-Side Fix)

Nagle's algorithm prevents sender SWS by requiring that the sender always have at most one unacknowledged small segment:

```text
Nagle's rule: If you have data to send and there's unacknowledged data in flight,
wait until either:
1. You have a full MSS worth of data, OR
2. All outstanding data is ACKed

This coalesces small writes into larger segments.
```

```bash
# Nagle is enabled by default
# To verify, check TCP_NODELAY on sockets
ss -tin state established | grep nodelay
# If "nodelay" appears: Nagle is DISABLED for this socket

# To enable Nagle explicitly in application code
import socket
s = socket.socket()
# Nagle is ON by default - don't set TCP_NODELAY unless you need low latency
# s.setsockopt(socket.IPPROTO_TCP, socket.TCP_NODELAY, 0)  # Enable Nagle
```

## Clark's Algorithm (Receiver-Side Fix)

Clark's algorithm prevents receiver SWS by not advertising window updates until the receiver has enough buffer to hold either a full MSS or half the socket buffer:

```text
Clark's rule: Don't send window updates for small increments.
Only send a window update when:
1. The buffer space freed >= 1 MSS (typically 1460 bytes), OR
2. The buffer space freed >= half the maximum receive window

Linux implements this automatically - you don't need to configure it.
```

## Checking for SWS in Application Code

```python
# SWS from sender side: making many small write() calls
import socket

# BAD: many small writes = many small TCP segments (if Nagle disabled)
s = socket.socket()
s.setsockopt(socket.IPPROTO_TCP, socket.TCP_NODELAY, 1)  # Nagle disabled!
for i in range(1000):
    s.send(b'x')   # Each call = separate TCP segment (SWS!)

# GOOD: buffer writes, send in larger chunks
buf = b''
for i in range(1000):
    buf += b'x'
    if len(buf) >= 1460:  # Full MSS
        s.send(buf)
        buf = b''
if buf:
    s.send(buf)
```

## When to Disable Nagle

```python
# Low-latency interactive applications (SSH terminal, game servers, trading)
# benefit from disabling Nagle because small packets should go immediately

s = socket.socket()
s.setsockopt(socket.IPPROTO_TCP, socket.TCP_NODELAY, 1)  # Disable Nagle

# Trade-off: Lower latency but more small packets on the network
# For bulk transfers: keep Nagle enabled (better efficiency)
# For interactive/real-time: disable Nagle for lower latency
```

## Conclusion

Silly Window Syndrome is an efficiency problem where TCP sends packets that are mostly header with tiny payloads. Nagle's algorithm (enabled by default) prevents sender SWS by coalescing small writes. Clark's algorithm in the kernel prevents receiver SWS by holding back small window updates. Only disable Nagle's algorithm for interactive applications that need sub-millisecond responsiveness - the cost is more small packets on the network.
