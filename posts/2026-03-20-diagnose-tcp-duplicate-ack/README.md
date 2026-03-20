# How to Diagnose TCP Duplicate ACK Issues

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: TCP, Duplicate ACK, Networking, Troubleshooting, Wireshark, SACK

Description: Understand why TCP duplicate ACKs are generated, how they signal packet loss or reordering, and how to use them to diagnose network problems.

## Introduction

A TCP duplicate ACK is sent when the receiver gets a packet that is out of order — the expected sequence number hasn't arrived yet, but a later one has. The receiver acknowledges the last in-order segment again (duplicate). Three consecutive duplicate ACKs trigger fast retransmit. While normal in small quantities, excessive duplicate ACKs signal consistent packet loss or reordering that needs investigation.

## When Duplicate ACKs are Generated

```
Normal:
Sender: 1, 2, 3, 4, 5 (in order)
ACK:    1, 2, 3, 4, 5 (no duplicates)

Packet 3 lost:
Sender: 1, 2,    4, 5
ACK:    1, 2, 2, 2, 2  ← dup ACKs for 2 (receiver waiting for 3)
                       ^^ 3 dup ACKs triggers fast retransmit of packet 3
```

## Capturing Duplicate ACKs

```bash
# Capture to file for analysis
tcpdump -i eth0 -n -w /tmp/dupacks.pcap 'tcp and host 10.20.0.5'

# Real-time detection of dup ACK sequences
tcpdump -i eth0 -n 'tcp and host 10.20.0.5' | \
  awk '
    /Flags \[.\].*ack/ {
      match($0, /ack ([0-9]+)/, a)
      if (a[1] == last_ack) {
        count++
        printf "Dup ACK %d for ack=%s\n", count, a[1]
      } else {
        count = 0
        last_ack = a[1]
      }
    }
  '
```

## Wireshark Analysis

```
# Show all duplicate ACKs
tcp.analysis.duplicate_ack

# Show ACKs with duplicate count >= 3 (loss-triggering)
tcp.analysis.duplicate_ack_num >= 3

# Combined view: dup ACKs and the retransmit they triggered
tcp.analysis.duplicate_ack or tcp.analysis.fast_retransmission

# Expert Information provides automatic analysis:
# Analyze → Expert Information
# Look for "Duplicate ACK" entries
```

## Counting Dup ACKs in a Capture

```bash
# Count all duplicate ACK events
tshark -r /tmp/dupacks.pcap \
  -Y "tcp.analysis.duplicate_ack" \
  -T fields -e frame.number -e tcp.ack 2>/dev/null | wc -l

# Show the most common ACK values for dup ACKs (identifies lost segments)
tshark -r /tmp/dupacks.pcap \
  -Y "tcp.analysis.duplicate_ack" \
  -T fields -e tcp.ack 2>/dev/null | sort | uniq -c | sort -rn | head -10
```

## Differentiating Loss from Reordering

```bash
# Loss: dup ACKs followed by a fast retransmit
# Reordering: dup ACKs followed by the expected packet arriving without retransmit

# In Wireshark:
# If dup ACK sequence is followed by tcp.analysis.fast_retransmission: loss
# If dup ACK sequence ends when out-of-order packet arrives: reordering

# Check kernel counters
nstat | grep -E "OFOQueue|FastRetrans|SpuriousRtx"
# High OFOQueue + low FastRetrans = mostly reordering
# High FastRetrans = actual loss
```

## Kernel Duplicate ACK Statistics

```bash
# Duplicate ACKs sent (as receiver)
nstat | grep TcpOutSegs   # total output
# Linux doesn't expose dup ACK count directly via nstat

# Proxy metrics (dup ACKs are a prerequisite for fast retransmits)
nstat | grep TcpExtTCPFastRetrans
# High FastRetrans = many 3-dup-ACK events = many loss events

# Watch for change rate
watch -n 2 "nstat -z | grep FastRetrans"
```

## Conclusion

Duplicate ACKs are the TCP receiver's way of saying "something arrived out of order." Three consecutive dup ACKs trigger fast retransmit — this is the primary loss detection mechanism. Occasional dup ACKs are normal in any network with slight reordering. Consistent 3-dup-ACK sequences followed by retransmissions confirm packet loss. Use Wireshark's `tcp.analysis.duplicate_ack_num >= 3` filter to count actual loss-triggering events, and distinguish them from simple reordering by checking whether a retransmission follows.
