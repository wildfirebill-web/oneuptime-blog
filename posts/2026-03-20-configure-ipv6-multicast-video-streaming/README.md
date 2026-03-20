# How to Configure IPv6 Multicast for Video Streaming

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Multicast, Video Streaming, IPTV, Networks

Description: A practical guide to configuring IPv6 multicast for video streaming deployments, including sender configuration, receiver setup, and network optimization.

## IPv6 Multicast for Video: Overview

IPv6 multicast is ideal for video streaming because:
- One stream serves unlimited viewers (no per-viewer bandwidth)
- IPv6's large address space provides ample multicast group addresses
- PIM-SSM provides source authentication for content protection

## Choosing the Right Multicast Approach

For video streaming:
- **PIM-SSM** (Source-Specific Multicast): Recommended - receivers specify exact source, prevents unauthorized streams
- **PIM-SM** with RP: Use when source isn't known ahead of time
- **SSM range**: Use `ff3e::/32` (global scope) or `ff3x::/32` with appropriate scope

## Setting Up a Video Stream Sender

```python
#!/usr/bin/env python3
# multicast_video_sender.py

# Send a video stream over IPv6 multicast

import socket
import time
import struct

class IPv6MulticastSender:
    def __init__(self, group_addr, port, interface, ttl=64):
        self.group_addr = group_addr
        self.port = port

        # Create UDP socket
        self.sock = socket.socket(socket.AF_INET6, socket.SOCK_DGRAM)

        # Set multicast hop limit (like TTL for IPv4)
        self.sock.setsockopt(socket.IPPROTO_IPV6,
                             socket.IPV6_MULTICAST_HOPS, ttl)

        # Set outgoing interface for multicast
        ifindex = socket.if_nametoindex(interface)
        self.sock.setsockopt(socket.IPPROTO_IPV6,
                             socket.IPV6_MULTICAST_IF, ifindex)

        # Don't receive your own multicast
        self.sock.setsockopt(socket.IPPROTO_IPV6,
                             socket.IPV6_MULTICAST_LOOP, 0)

    def send_frame(self, data):
        """Send a video frame to the multicast group"""
        self.sock.sendto(data, (self.group_addr, self.port))

# Usage: stream to group ff3e::iptv:channel1
sender = IPv6MulticastSender(
    group_addr='ff3e::db8:iptv:1',
    port=5004,
    interface='eth0',
    ttl=64
)

# Simulate sending video frames at 30fps
while True:
    frame_data = b'\x00' * 1316  # MPEG-TS packet size
    sender.send_frame(frame_data)
    time.sleep(1/30)  # 30fps
```

## Setting Up a Video Stream Receiver

```python
#!/usr/bin/env python3
# multicast_video_receiver.py

import socket
import struct

class IPv6MulticastReceiver:
    def __init__(self, group_addr, source_addr, port, interface):
        self.sock = socket.socket(socket.AF_INET6, socket.SOCK_DGRAM)
        self.sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        self.sock.bind(('', port))

        # Join the SSM (Source-Specific Multicast) group
        # This ensures we only receive from the authorized source
        ifindex = socket.if_nametoindex(interface)

        group_bytes = socket.inet_pton(socket.AF_INET6, group_addr)
        source_bytes = socket.inet_pton(socket.AF_INET6, source_addr)

        # struct sockaddr_in6
        def make_sa_in6(addr):
            return struct.pack('HHI16sI', socket.AF_INET6, 0, 0, addr, 0)

        # struct group_source_req
        mreq = struct.pack('II', ifindex, 0)
        mreq += make_sa_in6(group_bytes)
        mreq += make_sa_in6(source_bytes)

        self.sock.setsockopt(socket.IPPROTO_IPV6,
                             socket.MCAST_JOIN_SOURCE_GROUP, mreq)

        print(f"Joined SSM group ({source_addr}, {group_addr})")

    def receive_frame(self):
        data, addr = self.sock.recvfrom(65536)
        return data, addr

# Usage: receive IPTV channel 1 from authorized source
receiver = IPv6MulticastReceiver(
    group_addr='ff3e::db8:iptv:1',
    source_addr='2001:db8::stream-server',  # SSM: specify source
    port=5004,
    interface='eth0'
)

frame_count = 0
while True:
    data, addr = receiver.receive_frame()
    frame_count += 1
    if frame_count % 30 == 0:  # Log every second (30fps)
        print(f"Received frame {frame_count} ({len(data)} bytes) from {addr[0]}")
```

## Network Configuration for Video Multicast

```bash
# On routers: ensure PIM-SSM is configured for the stream group
vtysh
configure terminal

# Enable PIM on interfaces
interface eth0
 ipv6 pim

interface eth1
 ipv6 pim

# Enable MLD for receiver queries
interface eth0
 ipv6 mld version 2
 ipv6 mld query-interval 30

end

# Verify multicast route for the stream
vtysh -c "show ipv6 mroute ff3e::db8:iptv:1"
```

## Setting MTU for Video Streams

```bash
# Check MTU on multicast path (important for HD video)
# Default MTU 1500 may cause fragmentation

# Set stream MTU to safe value
ip link set eth0 mtu 9000  # Jumbo frames for high-bandwidth video (if supported)

# Or reduce video packet size to avoid fragmentation
# MPEG-TS natural size: 7 packets × 188 bytes = 1316 bytes
# Well under 1480 (IPv6 MTU for GigE with 20-byte header headroom)

# Test if large packets reach destination without fragmentation
ping6 -M do -s 1400 2001:db8::receiver
```

## Quality Monitoring for IPv6 Multicast Video

```bash
#!/bin/bash
# monitor_stream.sh - Monitor multicast video stream quality

MCAST_GROUP="ff3e::db8:iptv:1"
INTERFACE="eth0"
INTERVAL=10

echo "Monitoring $MCAST_GROUP on $INTERFACE every ${INTERVAL}s"

while true; do
    # Count packets received in the interval
    PACKETS_START=$(cat /proc/net/ip6_mr_cache | grep "$MCAST_GROUP" | awk '{print $7}')
    sleep $INTERVAL
    PACKETS_END=$(cat /proc/net/ip6_mr_cache | grep "$MCAST_GROUP" | awk '{print $7}')

    if [ -n "$PACKETS_START" ] && [ -n "$PACKETS_END" ]; then
        RATE=$(( (PACKETS_END - PACKETS_START) / INTERVAL ))
        echo "$(date): $MCAST_GROUP - $RATE packets/sec"
    else
        echo "$(date): $MCAST_GROUP - No multicast route (stream may be down)"
    fi
done
```

## Summary

IPv6 multicast video streaming uses UDP datagrams sent to a multicast group address. Use PIM-SSM with `ff3e::/32` addresses for secure streaming with source verification. The sender configures `IPV6_MULTICAST_HOPS` and `IPV6_MULTICAST_IF`; receivers use `MCAST_JOIN_SOURCE_GROUP` for SSM joins. Ensure routers have PIM and MLDv2 configured, and monitor stream health by tracking packet rates in the multicast forwarding table.
