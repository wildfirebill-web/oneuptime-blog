# How to Analyze TCP Window Size with Wireshark

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Wireshark, TCP, Networking, Packet Analysis, Performance, Troubleshooting

Description: Learn how to analyze TCP window size, window scaling, and zero-window events in Wireshark to diagnose throughput and performance issues.

---

The TCP receive window controls how much data can be in-flight before an acknowledgment is required. A shrinking or zero window is one of the most common causes of slow TCP throughput. Wireshark makes it easy to spot window-related performance problems.

---

## Display Filters for TCP Window Analysis

```
# Show all TCP traffic
tcp

# Show zero window packets (receiver buffer full)
tcp.window_size == 0

# Show window update packets
tcp.flags.ack == 1 and tcp.window_size > 0 and tcp.len == 0

# Show TCP window full packets
tcp.analysis.window_full

# Show all TCP expert info items
tcp.analysis.flags
```

---

## Enable TCP Window Scaling Dissection

TCP window scaling extends the window beyond 65535 bytes. Wireshark automatically applies the scale factor from the SYN/SYN-ACK exchange.

```
# Verify window scaling options during handshake
tcp.options.wscale

# Check actual scaled window size in packet details pane:
# Transmission Control Protocol
#   Window: 1024
#   Calculated window size: 131072  ← scaled value
#   Window size scaling factor: 128
```

---

## Visualize Window Size Over Time

1. Select a TCP stream packet.
2. Right-click → **Follow** → **TCP Stream** to isolate the stream.
3. Open **Statistics** → **TCP Stream Graphs** → **Window Scaling**.
4. The graph shows window size over time — a plateau near zero indicates a bottleneck.

---

## Common TCP Window Problems

| Symptom                  | Wireshark Filter                  | Cause                                |
|--------------------------|-----------------------------------|--------------------------------------|
| Zero window              | `tcp.analysis.zero_window`        | Receiver buffer exhausted            |
| Window full              | `tcp.analysis.window_full`        | Sender hit the receiver window limit |
| Zero window probe        | `tcp.analysis.zero_window_probe`  | Sender probing to detect recovery    |
| Keep-alive with no data  | `tcp.analysis.keep_alive`         | Idle connection maintenance          |

---

## Inspect Window Size in Packet Details

Click on a TCP packet and expand the **Transmission Control Protocol** section:

```
Transmission Control Protocol
  Source Port: 443
  Destination Port: 52400
  Window: 502
  Calculated window size: 128512
  Window size scaling factor: 256
```

A `Calculated window size` near 0 while data is queued indicates the receiver cannot keep up.

---

## Summary

Use `tcp.analysis.zero_window` and `tcp.analysis.window_full` filters to quickly locate TCP throughput bottlenecks in Wireshark. The TCP Stream Graph → Window Scaling view provides a visual timeline of window size changes. Correlate zero-window events with high latency or retransmissions to pinpoint whether the bottleneck is receiver-side buffer exhaustion or network congestion.
