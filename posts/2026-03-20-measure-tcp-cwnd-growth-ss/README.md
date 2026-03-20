# How to Measure TCP Congestion Window Growth with ss

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: TCP, Linux, ss, CWND, Congestion Control, Performance

Description: Use the ss command to measure and track TCP congestion window growth during active transfers, providing insight into congestion control behavior.

## Introduction

The `ss` (socket statistics) command on Linux provides detailed per-connection TCP information including the congestion window (CWND) size, smoothed RTT, and retransmission counters. This makes it an essential tool for understanding how congestion control algorithms behave during real transfers, without needing to capture packets.

## Key TCP Fields in ss Output

```bash
# Show detailed TCP info for all established connections
ss -tin state established

# Example output (annotated):
# tcp ESTAB 0 0 192.168.1.10:52341 10.20.0.5:8080
#  cubic wscale:7,7               ← congestion algorithm, window scale
#  rto:204                         ← current RTO in ms
#  rtt:1.2/0.5                    ← smoothed RTT / variance in ms
#  ato:40                          ← ACK timeout in ms
#  mss:1460                        ← MSS in bytes
#  pmtu:1500                       ← Path MTU
#  rcvmss:1460                     ← Receive MSS
#  advmss:1460                     ← Advertised MSS to peer
#  cwnd:10                         ← Congestion window in MSS units!
#  ssthresh:128                    ← Slow start threshold
#  bytes_sent:14600                ← Total bytes sent
#  bytes_acked:14600               ← Total bytes acknowledged
#  bytes_received:5840             ← Total bytes received
#  segs_out:10                     ← Segments sent
#  segs_in:5                       ← Segments received
#  data_segs_out:10                ← Data segments sent
#  send 96.5Mbps                   ← Calculated send rate
#  rcv_space:87380                 ← Receive buffer allocated
#  retrans:0/0                     ← Retransmitted/total retrans
```

## Capturing CWND Growth Over Time

```bash
#!/bin/bash
# Capture CWND values every 100ms during a transfer

TARGET="10.20.0.5"
OUTPUT="/tmp/cwnd_growth.csv"
echo "timestamp,cwnd_mss,cwnd_bytes,rtt_ms,retrans" > $OUTPUT

# Start a transfer in background
iperf3 -c $TARGET -t 30 &>/dev/null &
IPERF_PID=$!

# Capture CWND data
for i in $(seq 1 300); do  # 300 samples × 100ms = 30 seconds
    DATA=$(ss -tin state established "( dst $TARGET )" 2>/dev/null | \
           awk '/cwnd/{
             match($0, /cwnd:([0-9]+)/, c)
             match($0, /rtt:([0-9.]+)/, r)
             match($0, /mss:([0-9]+)/, m)
             match($0, /retrans:([0-9]+)\//, ret)
             if(c[1]) print c[1], c[1]*m[1]+0, r[1]+0, ret[1]+0
           }')
    if [ -n "$DATA" ]; then
        echo "$(date +%s.%3N),$DATA" | tr ' ' ',' >> $OUTPUT
    fi
    sleep 0.1
done

kill $IPERF_PID 2>/dev/null
echo "Data saved to $OUTPUT"

# Quick analysis
awk -F, 'NR>1{sum+=$2; n++; if($2>max)max=$2; if($2<min||min=="")min=$2}
  END{print "CWND - Min:", min, "Max:", max, "Avg:", sum/n" MSS"}' $OUTPUT
```

## Extracting RTT and CWND Together

```bash
# Show both RTT and CWND for quick assessment
ss -tin state established | \
  awk '/cwnd/{
    match($0, /rtt:([0-9.]+)/, rtt)
    match($0, /cwnd:([0-9]+)/, cwnd)
    match($0, /ssthresh:([0-9]+)/, thresh)
    printf "CWND: %s MSS, RTT: %s ms, ssthresh: %s\n", cwnd[1], rtt[1], thresh[1]
  }'
```

## Analyzing the Data

```bash
# After collecting cwnd_growth.csv, analyze the growth pattern:

python3 << 'EOF'
import csv

with open('/tmp/cwnd_growth.csv') as f:
    rows = list(csv.DictReader(f))

if rows:
    print(f"Samples: {len(rows)}")
    cwnd_values = [int(r['cwnd_mss']) for r in rows if r['cwnd_mss']]
    print(f"Min CWND: {min(cwnd_values)} MSS")
    print(f"Max CWND: {max(cwnd_values)} MSS")
    print(f"Final CWND: {cwnd_values[-1]} MSS")

    # Detect congestion events (CWND drops)
    drops = [(i, cwnd_values[i], cwnd_values[i-1])
             for i in range(1, len(cwnd_values))
             if cwnd_values[i] < cwnd_values[i-1] * 0.7]
    print(f"Congestion events (>30% CWND drop): {len(drops)}")
EOF
```

## Conclusion

`ss -tin` provides a comprehensive view of TCP connection internals without packet capture overhead. The CWND field in MSS units, combined with MSS size and RTT, tells you the throughput a connection is achieving. Track CWND over time to see congestion events as drops, and compare ssthresh against CWND to understand when slow start ends and congestion avoidance begins.
