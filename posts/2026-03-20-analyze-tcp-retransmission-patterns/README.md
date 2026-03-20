# How to Analyze TCP Retransmission Rates and Patterns

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: TCP, Retransmission, Networking, Analysis, Wireshark, Performance

Description: Analyze TCP retransmission patterns to distinguish between fast retransmits from loss events, spurious retransmits from reordering, and timeout retransmits from severe congestion.

## Introduction

Not all TCP retransmissions are equal. Fast retransmits (triggered by 3 dup ACKs) represent mild loss quickly recovered. Spurious retransmits occur when packets arrive out of order and the sender re-sends unnecessarily. Timeout retransmits indicate severe packet loss or congestion. Distinguishing these patterns tells you how serious the problem is and where to look.

## Types of TCP Retransmissions

| Type | Trigger | Severity | CWND Action |
|---|---|---|---|
| Fast Retransmit | 3 duplicate ACKs | Mild | CWND ÷ 2 |
| SACK Retransmit | SACK blocks show gaps | Mild | CWND ÷ 2 |
| Spurious Retransmit | Reordering, not loss | False positive | CWND restored |
| Timeout Retransmit | RTO expires | Severe | CWND = 1 MSS |

## Counting Each Type in Kernel

```bash
# Get all retransmission-related counters
nstat -a | grep -iE "retrans|timeout|spurious|sack" | grep -v "^#"

# Key counters:
# TcpRetransSegs: Total retransmitted segments (all types)
# TcpExtTCPFastRetrans: Fast retransmissions (3 dup ACKs)
# TcpExtTCPSlowStartRetrans: Timeout retransmissions (most severe)
# TcpExtTCPSpuriousRtxHostQueues: Spurious retransmits detected
# TcpExtTCPSackFailures: SACK retransmit failures

# Calculate what percentage are severe (timeout) vs mild (fast)
FAST=$(nstat -z 2>/dev/null | awk '/TcpExtTCPFastRetrans/{print $2+0}')
SLOW=$(nstat -z 2>/dev/null | awk '/TcpExtTCPSlowStartRetrans/{print $2+0}')
TOTAL=$(nstat -z 2>/dev/null | awk '/TcpRetransSegs/{print $2+0}')
echo "Fast retransmits: $FAST"
echo "Timeout retransmits: $SLOW"
echo "Total: $TOTAL"
```

## Monitoring Retransmission Rate

```bash
#!/bin/bash
# Track retransmission rate over time

PREV_RETRANS=0
PREV_SEGS=0

while true; do
    RETRANS=$(nstat -z 2>/dev/null | awk '/TcpRetransSegs/{print $2+0}')
    SEGS=$(nstat -z 2>/dev/null | awk '/TcpOutSegs/{print $2+0}')

    DELTA_RETRANS=$((RETRANS - PREV_RETRANS))
    DELTA_SEGS=$((SEGS - PREV_SEGS))

    if [ $DELTA_SEGS -gt 0 ]; then
        RATE=$(echo "scale=2; $DELTA_RETRANS * 100 / $DELTA_SEGS" | bc)
        echo "$(date +%H:%M:%S) Retransmit rate: $RATE% ($DELTA_RETRANS/$DELTA_SEGS)"
    fi

    PREV_RETRANS=$RETRANS
    PREV_SEGS=$SEGS
    sleep 5
done
```

## Wireshark Retransmission Analysis

```
# In Wireshark:

# All retransmissions (any type)
tcp.analysis.retransmission

# Fast retransmits (triggered by dup ACKs - mild)
tcp.analysis.fast_retransmission

# Spurious retransmissions (send-then-receive-before-RTO)
tcp.analysis.spurious_retransmission

# View retransmit statistics:
# Statistics → TCP Stream Graphs → Time-Sequence (Stevens)
# Retransmissions appear as data points below the main line

# Expert Information shows all retransmit events:
# Analyze → Expert Information → filter by "Retransmission"
```

## Interpreting Patterns

```bash
# Pattern 1: Mostly fast retransmits (< 1% of segments)
# → Normal behavior, occasional loss with quick recovery
# → No action needed if throughput is acceptable

# Pattern 2: Mostly timeout retransmits
# → Severe congestion or unreliable link
# → Investigate: check interface errors, reduce traffic, check routing

# Pattern 3: Many spurious retransmits
# → High packet reordering (ECMP, satellite, wireless)
# → Fix: increase tcp_reordering tolerance
sysctl -w net.ipv4.tcp_reordering=6

# Pattern 4: Periodic retransmit spikes
# → Bursty cross-traffic causing momentary congestion
# → Fix: fair queuing, traffic shaping
```

## Conclusion

TCP retransmission analysis is most useful when you classify by type rather than counting all retransmissions together. A 1% fast retransmit rate is acceptable; any timeout retransmissions are concerning. High spurious retransmit rates indicate reordering, not true loss — increasing `tcp_reordering` tolerance prevents unnecessary CWND reductions. Monitor continuously with the rate script to detect when retransmissions start increasing.
