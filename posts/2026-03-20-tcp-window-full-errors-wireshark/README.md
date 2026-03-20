# How to Diagnose TCP Window Full Errors in Wireshark

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: TCP, Wireshark, Window Full, Networking, Performance, Troubleshooting

Description: Identify and diagnose TCP Window Full events in Wireshark that indicate the receiver's buffer is saturated, causing the sender to pause transmission.

## Introduction

"TCP Window Full" appears in Wireshark when the sender has filled the receiver's advertised window and must stop sending until it receives an ACK with an updated (larger) window. This is a flow control mechanism — not an error — but frequent window-full events indicate a throughput bottleneck caused by either a slow receiver or undersized buffers.

## What TCP Window Full Means

```
Sender is limited:
Sender can have at most [receiver's window size] unacknowledged bytes in flight.

When window fills:
1. Sender sends data until unacked bytes = receiver's window
2. Sender STOPS (pauses)
3. Receiver processes data, frees buffer space
4. Receiver sends ACK with updated window
5. Sender resumes

If receiver processes slowly: sender pauses frequently → low throughput
If receiver buffer is small: window fills quickly → many pauses
```

## Finding Window Full Events in Wireshark

```
# Wireshark display filter for Window Full events
tcp.analysis.window_full

# Also look for Zero Window events (receiver window = 0)
tcp.analysis.zero_window

# Combined view
tcp.analysis.window_full or tcp.analysis.zero_window
```

## Using tcpdump to Spot Window Saturation

```bash
# Capture and look for small window advertisements
tcpdump -i eth0 -n -v 'tcp and host 10.20.0.5' | grep "win [0-9]" | \
  awk '{
    match($0, /win ([0-9]+)/, a)
    if (a[1]+0 < 1000) print "SMALL WINDOW:", $0
  }'

# A rapidly shrinking window in sequential packets indicates receiver saturation
```

## Diagnosing the Root Cause

### Cause 1: Slow Application (Receiver Not Reading Fast Enough)

```bash
# Check receiver's application CPU usage
top -p $(pgrep myapp)

# Check if application has a slow processing loop
# Profile with strace to see if recv() is being called
strace -p $(pgrep myapp) -e trace=recv,recvfrom,recvmsg 2>&1 | head -20

# If recv() calls are slow or infrequent: application bottleneck
```

### Cause 2: Small Receive Buffer

```bash
# Check current receive buffer sizes
sysctl net.ipv4.tcp_rmem

# If default is 128KB and your BDP is larger, buffer is the bottleneck
# Increase max receive buffer
sysctl -w net.ipv4.tcp_rmem="4096 262144 16777216"
```

### Cause 3: Memory Pressure

```bash
# Check if system has memory available for socket buffers
free -m
cat /proc/net/sockstat | grep TCP

# If system is memory-constrained, kernel limits socket buffer sizes
# Add memory or reduce other applications' memory usage
```

## Wireshark Timeline Analysis

```
In Wireshark's TCP stream graph (Statistics → TCP Stream Graphs):
1. Open "Time-Sequence (Stevens)" graph
2. Flat sections (no new sequence numbers advancing) = Window Full pauses
3. Long flat sections = severe window starvation

Timeline patterns:
- Regular flat sections: periodic window stalls (application processing delay)
- Rare flat sections: normal TCP flow control
- Long flat section followed by burst: Zero Window recovery
```

## Conclusion

TCP Window Full events are flow control — not errors — but they directly reduce throughput. A sender pausing because the window is full means data is arriving at the receiver faster than it can be consumed or buffered. The fix is either increasing receive buffer sizes (for buffer bottlenecks) or optimizing the receiving application (for processing bottlenecks). Wireshark's `tcp.analysis.window_full` filter makes it easy to quantify how often this is happening.
