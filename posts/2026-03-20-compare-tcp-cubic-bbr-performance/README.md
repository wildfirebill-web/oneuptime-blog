# How to Compare TCP CUBIC and BBR Performance

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: TCP, BBR, CUBIC, Performance, Networking, Linux

Description: Systematically compare TCP CUBIC and BBR performance on your network using iperf3 benchmarks and Wireshark analysis to determine the best algorithm for your workload.

## Introduction

CUBIC and BBR take fundamentally different approaches to congestion control. CUBIC reacts to packet loss; BBR models bandwidth and RTT directly. The performance difference depends heavily on network characteristics — CUBIC can outperform BBR on lossless LAN, while BBR dramatically outperforms CUBIC on lossy or high-latency paths.

## Setting Up the Comparison

```bash
# Install iperf3 on both server and client
apt install iperf3

# Server side
iperf3 -s --daemon

# Ensure both CUBIC and BBR are available
modprobe tcp_bbr
sysctl net.ipv4.tcp_available_congestion_control
```

## Benchmark Script

```bash
#!/bin/bash
# Compare CUBIC vs BBR under different network conditions

SERVER="10.20.0.5"
DURATION=30
RESULTS_FILE="/tmp/congestion_comparison.txt"

run_test() {
    local algo=$1
    local desc=$2
    sysctl -w net.ipv4.tcp_congestion_control=$algo >/dev/null 2>&1
    RESULT=$(iperf3 -c $SERVER -t $DURATION -J 2>/dev/null | \
             python3 -c "
import sys, json
d = json.load(sys.stdin)
sent_bps = d['end']['sum_sent']['bits_per_second']
retr = d['end']['sum_sent']['retransmits']
print(f'{sent_bps/1e6:.1f}Mbps retrans={retr}')
")
    echo "$algo ($desc): $RESULT" | tee -a $RESULTS_FILE
}

echo "=== Without Network Impairment ===" | tee $RESULTS_FILE
run_test cubic "clean LAN"
run_test bbr "clean LAN"

echo "=== With 50ms RTT ===" | tee -a $RESULTS_FILE
tc qdisc add dev eth0 root netem delay 50ms
run_test cubic "50ms RTT"
run_test bbr "50ms RTT"
tc qdisc del dev eth0 root

echo "=== With 100ms RTT + 1% Loss ===" | tee -a $RESULTS_FILE
tc qdisc add dev eth0 root netem delay 100ms loss 1%
run_test cubic "100ms+1%loss"
run_test bbr "100ms+1%loss"
tc qdisc del dev eth0 root

cat $RESULTS_FILE
```

## Typical Results

```
Clean LAN (< 5ms RTT, 0% loss):
  CUBIC: 950 Mbps  retrans=2
  BBR:   940 Mbps  retrans=0
  → Approximately equal, CUBIC slightly ahead

50ms RTT, 0% loss:
  CUBIC: 180 Mbps  retrans=5
  BBR:   750 Mbps  retrans=0
  → BBR significantly better

100ms RTT + 1% loss:
  CUBIC: 25 Mbps   retrans=120
  BBR:   650 Mbps  retrans=8
  → BBR dramatically better
```

## Latency Under Load Comparison

```bash
# Check if algorithm causes bufferbloat (latency spike during transfer)
# Terminal 1: ping to measure latency
ping -c 60 -i 1 $SERVER > /tmp/ping_during_transfer.txt &

# Terminal 2: run iperf3 with each algorithm
sysctl -w net.ipv4.tcp_congestion_control=cubic
iperf3 -c $SERVER -t 30

# Check latency during CUBIC:
grep -oP 'time=\K[\d.]+' /tmp/ping_during_transfer.txt | awk '{sum+=$1;n++}END{print sum/n"ms avg"}'

# Repeat with BBR
sysctl -w net.ipv4.tcp_congestion_control=bbr
iperf3 -c $SERVER -t 30
```

## Conclusion

BBR consistently outperforms CUBIC on any path with measurable latency or packet loss. For pure LAN with sub-millisecond RTT and zero loss, CUBIC is adequate. The recommendation for internet-facing services is BBR with `net.core.default_qdisc=fq`. Run the benchmark script in your own environment to confirm results — network characteristics vary significantly and the performance winner depends on your actual path.
