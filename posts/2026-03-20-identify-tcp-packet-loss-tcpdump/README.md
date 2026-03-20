# How to Identify TCP Packet Loss with tcpdump

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: TCP, tcpdump, Packet Loss, Networking, Troubleshooting, Analysis

Description: Use tcpdump to capture TCP traffic and identify packet loss through sequence number gaps, duplicate ACKs, and retransmission patterns.

## Introduction

tcpdump is the ground truth for TCP packet analysis. By examining sequence numbers and ACK numbers in a capture, you can identify exactly which packets were lost, when loss occurred, and whether the connection recovered through fast retransmit or had to wait for timeout. This is the most precise way to confirm and quantify packet loss on a specific connection.

## Basic Capture for Loss Analysis

```bash
# Capture TCP traffic with a specific host

tcpdump -i eth0 -n -w /tmp/loss_analysis.pcap 'tcp and host 10.20.0.5'

# Run during a period when loss is occurring
# Then analyze the capture

# Quick capture and analysis (no file)
tcpdump -i eth0 -n -S 'tcp and port 8080 and host 10.20.0.5' | head -50
# -S = absolute sequence numbers (easier to spot gaps)
```

## Identifying Packet Loss from Sequence Numbers

```bash
# Analyze a capture file for sequence number gaps
tshark -r /tmp/loss_analysis.pcap \
  -Y "ip.dst == 10.20.0.5" \
  -T fields \
  -e frame.number \
  -e tcp.seq \
  -e tcp.len \
  -e tcp.ack \
  | awk '
    prev_seq != "" && $2 != prev_seq + prev_len {
      print "GAP at frame " $1 ": expected seq " prev_seq+prev_len " got " $2
    }
    {prev_seq=$2; prev_len=$3}
  '
```

## Detecting Duplicate ACKs (Loss Signals)

```bash
# Duplicate ACKs indicate the receiver got out-of-order data (possible loss ahead)
tcpdump -r /tmp/loss_analysis.pcap -n | \
  awk '
    /Flags \[.\].*ack/ {
      match($0, /ack ([0-9]+)/, a)
      if (a[1] == last_ack) {
        dup_count++
        if (dup_count == 3) print "3 DUP ACKs for ack=" a[1] ": LOSS EVENT at " $1
      } else {
        dup_count = 0
      }
      last_ack = a[1]
    }
  '
```

## Wireshark Packet Loss Filters

```text
# In Wireshark:

# Find all retransmissions (after loss)
tcp.analysis.retransmission

# Find the duplicate ACKs that triggered fast retransmit
tcp.analysis.duplicate_ack_num >= 3

# Find sequence number gaps (evidence of loss)
tcp.analysis.lost_segment

# Combined loss event view
tcp.analysis.lost_segment or tcp.analysis.fast_retransmission or tcp.analysis.retransmission
```

## Automated Loss Detection Script

```bash
#!/bin/bash
# Capture and report packet loss statistics for a connection

TARGET="10.20.0.5"
PORT="8080"
DURATION=30

echo "Capturing $DURATION seconds of traffic to $TARGET:$PORT..."
tcpdump -i eth0 -n -w /tmp/loss_test.pcap \
  "tcp and host $TARGET and port $PORT" &
TCPDUMP_PID=$!

sleep $DURATION
kill $TCPDUMP_PID 2>/dev/null

echo "Analyzing capture..."
tshark -r /tmp/loss_test.pcap -q -z io,stat,1 2>/dev/null | head -20

# Count retransmission events
RETRANS=$(tshark -r /tmp/loss_test.pcap \
  -Y "tcp.analysis.retransmission" 2>/dev/null | wc -l)

TOTAL=$(tshark -r /tmp/loss_test.pcap 2>/dev/null | wc -l)

echo "Total packets: $TOTAL"
echo "Retransmissions: $RETRANS"
if [ $TOTAL -gt 0 ]; then
    echo "Retransmit rate: $(echo "scale=2; $RETRANS * 100 / $TOTAL" | bc)%"
fi
```

## Conclusion

tcpdump and tshark provide definitive evidence of TCP packet loss through sequence number gaps and duplicate ACK patterns. A single `tshark -Y "tcp.analysis.retransmission"` command on a capture file counts loss events. Combined with the sequence number timeline, you can determine when loss occurred, how much was lost, and whether the TCP connection recovered gracefully or required a timeout. This is the most reliable method for confirming suspected packet loss.
