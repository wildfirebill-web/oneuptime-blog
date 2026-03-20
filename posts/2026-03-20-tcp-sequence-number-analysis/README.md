# How to Use TCP Sequence Number Analysis for Debugging

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: TCP, Sequence Numbers, Debugging, Wireshark, Packet Analysis, Networking

Description: Use TCP sequence number analysis in Wireshark and tcpdump to identify retransmissions, reordering, gaps, and duplicate data in captured network traffic.

## Introduction

TCP sequence numbers track every byte sent on a connection. When something goes wrong — lost packets, reordering, retransmissions — the sequence number timeline reveals exactly what happened. Wireshark builds time-sequence graphs automatically; tcpdump lets you trace sequence numbers in the terminal. Both approaches turn raw packet captures into a clear narrative of connection behavior.

## Understanding Sequence Number Basics

```
Initial Sequence Number (ISN): random, chosen at SYN
Subsequent segments: ISN + bytes_sent_so_far
ACK number: "I've received up to byte N, send me N+1 next"

Example flow:
SYN:        seq=1000, len=0    → ISN is 1000
SYN-ACK:    seq=5000, ack=1001 → server ISN=5000, ACKs client's SYN
Data:       seq=1001, len=500  → bytes 1001-1500
ACK:        ack=1501           → "received through byte 1500"
Data:       seq=1001, len=500  → RETRANSMIT! Same seq sent again
```

## tcpdump Sequence Number Analysis

```bash
# Capture with sequence numbers shown
tcpdump -i eth0 -n -S host 10.20.0.5 and port 80
# -S shows absolute sequence numbers (easier for correlation)

# Show relative sequence numbers (default, easier to read)
tcpdump -i eth0 -n host 10.20.0.5 and port 80

# Output shows:
# Flags [S], seq 3232323, win 65535
# Flags [S.], seq 1234567, ack 3232324, win 65535
# Flags [.], ack 1234568
# Flags [P.], seq 1:501, ack 1, length 500   ← data: bytes 1-500

# Spot retransmissions: same seq number appearing twice
tcpdump -r capture.pcap -n | awk '{print $5}' | sort | uniq -d
# Duplicate sequence ranges = retransmissions
```

## Wireshark Time-Sequence Graph

```
In Wireshark:
1. Open capture file
2. Select a packet in the TCP stream
3. Statistics → TCP Stream Graphs → Time-Sequence (Stevens)

What to look for:
- Normal: smooth upward diagonal line (bytes increasing over time)
- Retransmission: vertical backtrack (same sequence resent)
- Zero window: horizontal flat line (no new data sent)
- Slow start: visible exponential growth at start
- Congestion event: sudden slope reduction after loss
```

## Identifying Problems from Sequence Numbers

```bash
# In Wireshark display filters:

# Show all retransmissions
tcp.analysis.retransmission

# Show out-of-order segments (receiver got higher seq before lower)
tcp.analysis.out_of_order

# Gaps in received sequence space (missing segments)
tcp.analysis.lost_segment

# Duplicate ACKs (receiver got out-of-order, sending dup ACKs)
tcp.analysis.duplicate_ack

# Combined: all TCP analysis events
tcp.analysis.flags
```

## Sequence Number Gaps and Reordering

```bash
# A gap in sequence numbers means:
# - Segment was lost (most common)
# - Segment arrived out of order (reordering)
# - Segment is still in transit

# Distinguish loss from reordering:
# Loss: gap is never filled → sender retransmits after 3 dup ACKs or RTO
# Reorder: gap fills in within a few milliseconds from a later packet

# Check for reordering with tcpdump:
tcpdump -r capture.pcap -n 'tcp and host 10.20.0.5' | \
  awk '/seq/{seq=$5; print NR, seq}' | sort -k2 -n | head -20
# If sequence numbers arrive out of numerical order: reordering
```

## Calculating Throughput from Sequence Numbers

```bash
# From a tcpdump capture, calculate actual data throughput:
tcpdump -r capture.pcap -n 'tcp and dst host 10.20.0.5 and dst port 80' | \
  awk 'NR==1{first=$1; start_seq=$8}
       {last=$1; end_seq=$8}
       END{
         duration=last-first
         bytes=(end_seq+0)-(start_seq+0)
         mbps=bytes*8/duration/1e6
         printf "Duration: %.2f sec, Bytes: %d, Throughput: %.2f Mbps\n",
           duration, bytes, mbps
       }'
```

## Conclusion

TCP sequence number analysis transforms "something is slow" into "packet at byte offset X was retransmitted at time T." Use `tcpdump -S` to trace absolute sequences in terminal. Use Wireshark's time-sequence graph for visual analysis. Duplicate sequence numbers always mean retransmission; gaps with subsequent fill-in mean reordering; gaps that stay = loss. This level of analysis makes it possible to pinpoint exactly which packets are being lost and when.
