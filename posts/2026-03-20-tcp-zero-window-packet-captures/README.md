# How to Interpret TCP Zero Window Events in Packet Captures

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: TCP, Wireshark, Zero Window, Networking, Flow Control, Performance

Description: Understand TCP Zero Window events in packet captures, what they indicate about receiver-side processing capacity, and how to resolve them.

## Introduction

A TCP Zero Window event occurs when the receiver's buffer is completely full and it advertises a window size of zero to the sender. The sender must stop transmitting completely and wait for a Zero Window Probe or a Window Update from the receiver. This is TCP's emergency brake — it completely halts the sender to give the receiver time to drain its buffer.

## What Happens During Zero Window

```
Normal: Receiver window = 32KB (sender can transmit up to 32KB ahead)
Zero Window: Receiver window = 0 (sender MUST STOP completely)

Sequence:
1. Receiver's buffer fills → advertises window=0
2. Sender stops sending data
3. Sender sends Zero Window Probe (1-byte payload) every few seconds
4. When receiver drains buffer: sends Window Update (window > 0)
5. Sender resumes transmission
```

## Finding Zero Window in Wireshark

```
# Filter for Zero Window events
tcp.analysis.zero_window

# Filter for Zero Window Probes (sender checking if receiver recovered)
tcp.analysis.zero_window_probe

# Filter for Window Updates (receiver recovered from Zero Window)
tcp.analysis.window_update

# Combined view of the full event sequence
tcp.analysis.zero_window or tcp.analysis.zero_window_probe or tcp.analysis.window_update
```

## Capturing Zero Window Events with tcpdump

```bash
# Capture and flag packets with window size 0
tcpdump -i eth0 -n -v 'tcp' 2>/dev/null | \
  awk '/win 0 /{print "ZERO WINDOW:", $0}'

# Or filter at the packet level
tcpdump -i eth0 -n 'tcp[14:2] == 0 and tcp[tcpflags] != tcp-syn'
# tcp[14:2] = window field; filter out SYN (which sometimes has window=0)
```

## Diagnosing the Root Cause

```bash
# Zero Window indicates the receiver application is NOT reading data fast enough
# The receive buffer fills up because data arrives faster than the app reads it

# Check the receiving application's CPU usage
top -p $(pgrep your-app)

# Check if the application has slow I/O (disk writes, DB queries)
iostat -xz 1 5
# If %util is high: disk I/O is causing the app to read sockets slowly

# Check for memory pressure limiting buffer allocation
free -m
cat /proc/sys/vm/pressure_cache

# Profile application socket reads
strace -p $(pgrep your-app) -e trace=recv,recvfrom -T 2>&1 | \
  awk '{match($0, /<([0-9.]+)>/, t); if(t[1]+0>0.01) print "SLOW RECV:", $0}'
```

## Duration Analysis

```bash
# In Wireshark: measure how long the Zero Window persisted
# 1. Filter: tcp.analysis.zero_window_probe
# 2. Note the timestamp of the first probe
# 3. Filter: tcp.analysis.window_update for the same stream
# 4. Calculate: window_update_time - first_zero_window_time = Zero Window duration

# Long Zero Window duration (>1 second) indicates serious receiver bottleneck
# Short Zero Window duration (<100ms) is acceptable in normal operation
```

## Fixes for Zero Window Issues

```bash
# Fix 1: Increase the application's receive rate
# (Profile and optimize the application's data processing)

# Fix 2: Increase receive buffer so it takes longer to fill
sysctl -w net.ipv4.tcp_rmem="4096 262144 16777216"

# Fix 3: For transfers: use dedicated transfer threads
# Don't mix processing and socket reading in the same thread

# Fix 4: Enable socket-level flow control in the application
# Use non-blocking sockets with select/poll/epoll
```

## Conclusion

TCP Zero Window is the receiver telling the sender "I'm overwhelmed, please stop." It's a symptom of the receiving application not reading data fast enough. Short Zero Window events are normal under brief bursts; long or frequent Zero Windows indicate a bottleneck in the receiving application's processing pipeline. Wireshark's `tcp.analysis.zero_window` filter makes it trivial to quantify how often and how long this is happening in a capture.
