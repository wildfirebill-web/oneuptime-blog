# How to Understand TCP Fast Recovery and Fast Retransmit

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: TCP, Fast Recovery, Fast Retransmit, Congestion Control, Networking

Description: Understand how TCP fast retransmit and fast recovery avoid waiting for timeout by using duplicate ACKs to detect and recover from packet loss quickly.

## Introduction

Before fast retransmit and fast recovery, a lost TCP segment required waiting for the full RTO (Retransmission Timeout) - typically hundreds of milliseconds to seconds. Fast retransmit detects loss earlier using three duplicate ACKs, and fast recovery allows the connection to continue sending at reduced rate during retransmission rather than restarting from scratch.

## Fast Retransmit: How It Works

```text
Normal operation:
Sender: segment 1, 2, 3, 4, 5
ACK:           1, 2,    4, 4, 4  ← segment 3 was lost

Receiver receives segment 4, but is missing segment 3
Receiver sends duplicate ACK for the last received (ACK=3)
After 3 duplicate ACKs of ACK=3:
  → Sender retransmits segment 3 immediately (don't wait for timeout!)
```

## Fast Recovery: Continued Transmission

```text
Without fast recovery (Reno original):
After loss: CWND = 1 MSS → restart slow start from scratch

With fast recovery:
After 3 dup ACKs:
  ssthresh = CWND / 2        (remember where we were)
  CWND = ssthresh + 3 MSS   (account for the 3 dup ACK segments)
  Continue sending (don't reset to slow start)
  When lost segment is ACKed: CWND = ssthresh (congestion avoidance mode)
```

## Capturing Fast Retransmit in tcpdump

```bash
# Capture to observe fast retransmit

tcpdump -i eth0 -n -w /tmp/fastretrans.pcap 'tcp and host 10.20.0.5'

# Analyze: look for same sequence number appearing twice
tshark -r /tmp/fastretrans.pcap -Y "tcp.analysis.fast_retransmission" \
  -T fields -e frame.time_relative -e ip.src -e tcp.seq
```

## Wireshark Filters

```text
# Fast retransmission (triggered by 3 dup ACKs)
tcp.analysis.fast_retransmission

# Duplicate ACKs that trigger fast retransmit
tcp.analysis.duplicate_ack

# View the sequence of dup ACKs leading to fast retransmit
tcp.analysis.duplicate_ack or tcp.analysis.fast_retransmission
```

## Monitoring Fast Retransmissions in Kernel

```bash
# Count fast retransmissions (less severe than timeout retransmissions)
nstat -a | grep -i "fast\|retrans"
# TcpExtTCPFastRetrans: fast retransmits triggered by dup ACKs
# TcpExtTCPSlowStartRetrans: timeout-triggered retransmits (more severe)
# TcpRetransSegs: total retransmissions

# High FastRetrans / total retransmissions ratio = good (fast detection)
# High SlowStartRetrans ratio = bad (using full timeout, very slow recovery)

watch -n 2 "nstat -z | grep -E 'FastRetrans|SlowStart|TcpRetrans'"
```

## Enabling SACK for Better Fast Recovery

```bash
# With SACK, the receiver can tell the sender exactly which segments arrived
# This allows more efficient retransmission (only missing segments)
sysctl net.ipv4.tcp_sack   # Should be 1

# Without SACK: sender retransmits from the first lost segment onward
# With SACK: sender retransmits only the specific lost segments
```

## Simulating Loss to Test Fast Retransmit

```bash
# Add 2% loss to simulate packet loss
tc qdisc add dev eth0 root netem loss 2%

# Run a transfer and watch for fast retransmits
iperf3 -c 10.20.0.5 -t 10 -i 1
# Look for "Retr" column - non-zero values indicate retransmissions

# Check kernel counters before and after
BEFORE=$(nstat -z | awk '/TcpExtTCPFastRetrans/{print $2}')
iperf3 -c 10.20.0.5 -t 10 &>/dev/null
AFTER=$(nstat -z | awk '/TcpExtTCPFastRetrans/{print $2}')
echo "Fast retransmissions: $((AFTER - BEFORE))"

tc qdisc del dev eth0 root
```

## Conclusion

Fast retransmit (3 dup ACKs) and fast recovery are fundamental to modern TCP performance. They allow TCP to detect and recover from individual packet loss in roughly one RTT instead of waiting for the full timeout. SACK enhances both by allowing selective retransmission. A healthy network should have mostly fast retransmissions (not timeout-based); seeing high slow-start retransmissions indicates severe congestion or loss requiring investigation.
